# Technology Deep Dive 4: LiteLLM Routing, Circuit Breaker FSM, Tenacity & `tiktoken`

## 1. Fundamental Concepts & Internals

### Unified LLM Mediation (LiteLLM)
- Each provider (Groq, OpenAI, Mistral, Anthropic) uses different SDKs, base URLs, auth headers, and error formats (`overloaded_error`, `rate_limit_exceeded`, `service_unavailable`).
- **LiteLLM** normalizes them behind an OpenAI-compatible `litellm.acompletion()` async interface (backed by `httpx.AsyncClient` connection pooling) and maps upstream errors into a unified Python exception hierarchy (`RateLimitError`, `ServiceUnavailableError`, `InternalServerError`, `APIConnectionError`, `Timeout`).

### Retries vs. Circuit Breakers (Complementary Roles)
- **Retries (Tenacity):** Handle *brief, isolated* transient blips (single packet drop, momentary 429). Fixed-interval retries synchronize client failures into **Thundering Herd** waves. **Full Jitter** draws wait times randomly across the full exponential ceiling:
  $$T_{\text{sleep}} \sim \text{Uniform}\left(0,\; \min(T_{\text{max}},\; \text{multiplier} \cdot 2^{\text{attempt}})\right)$$
  spreading client retries smoothly across time (AWS Architecture studies show Full Jitter minimizes server contention better than Equal or pure Exponential backoff).
- **Circuit Breaker (Michael Nygard FSM):** Handles *sustained* outages. Without a breaker, every incoming request waits `3 × ~4s` timeouts/retries before failing over—holding sockets open and saturating worker concurrency.
  - **`CLOSED`:** Normal traffic to primary; consecutive failures tracked; resets to `0` on success.
  - **`OPEN`:** At threshold (`5`), trips immediately. Subsequent calls return `False` from `can_execute()` in `0ms`, routing straight to fallback without hitting the broken provider.
  - **`HALF_OPEN`:** After `recovery_timeout` (`60s`), permits a probe to test recovery. Success $\to$ `CLOSED`; failure $\to$ back to `OPEN` for another `60s`.

### Byte-Pair Encoding (`tiktoken`)
- Word vocabularies fail on OOV/typos; character tokenization blows up Transformer $O(N^2)$ attention lengths.
- **BPE** starts from 256 raw byte tokens and iteratively merges the most frequent adjacent byte pairs into subwords (`"pgvector"` $\to$ `["pg", "vector"]`).
- **`tiktoken` Speed:** Written by OpenAI in Rust using compiled DFA regex pre-tokenizers (`regex` crate) and 64-bit hash merge maps—tokenizing at $10^6+$ tokens/sec without Python regex backtracking.

---

## 2. Why Selected for SentinelRAG
- **Thin Gateway vs. Heavy Frameworks (LangChain/LlamaIndex):** SentinelRAG avoids deep framework abstractions (`RunnableSequence`, opaque callbacks, rapid breaking minor releases) so on-call debugging sees crisp, direct stack traces and deterministic circuit-breaker transitions.
- **Ultra-Low TTFT + High Reliability:** Routes primary inference to Groq (`llama-3.3-70b-versatile`) for fast generation, failing over automatically to OpenAI (`gpt-4o-mini`) and Mistral (`mistral-small-latest`).
- **Exact Cost Attribution:** Rough `len(text)//4` heuristics drift by $\pm 30\%$ on JSON and code; `tiktoken` measures exact ChatML prompt + completion tokens.

---

## 3. Project Implementation & Configuration Details

### A. Coroutine-Safe Circuit Breaker (`app/core/circuit_breaker.py`)
- Uses `asyncio.Lock()` (never `threading.Lock()`, which blocks the entire OS thread and event loop!).
- Defaults from `app/core/config.py`: `failure_threshold = 5`, `recovery_timeout = 60.0s`, single-call `timeout = 4.0s`.
- **Important Subtlety to Know:** In `can_execute()`, once `OPEN` transitions to `HALF_OPEN` after `60s`, `can_execute()` returns `self._state == CircuitState.HALF_OPEN`. If 10 concurrent coroutines arrive in the exact microsecond while the single probe is still in-flight (before `record_success`/`record_failure` runs), `can_execute()` returns `True` for them too unless an explicit `_probe_in_flight` boolean guard is added! Mentioning this shows deep concurrency awareness.
- **Time Base:** Uses `time.time()` in `circuit_breaker.py` (whereas `time.monotonic()` is safer against NTP backward clock jumps—another sharp improvement point).

### B. Transient Error Filter & Tenacity Retryer (`app/services/resilience.py`)
- `is_transient_error(exc)` retries **only** `RateLimitError`, `ServiceUnavailableError`, `InternalServerError`, `APIConnectionError`, `Timeout`, `TimeoutError`, `ConnectionError`, `httpx.TimeoutException`, `httpx.NetworkError`, HTTP status codes `{429, 502, 503, 504}`, or matching message strings (`"rate limit"`, `"503"`, `"429"`, `"timed out"`, etc.).
- **Fail-Fast on Permanent Errors:** `400`, `401`, `403`, `404` return `False` and raise immediately—never burn retries on an invalid API key or malformed request.
- Uses `AsyncRetrying(..., wait=wait_random_exponential(multiplier=multiplier, max=max_wait), stop=stop_after_attempt(max_attempts), reraise=True)`.

### C. Multi-Provider Cascade (`app/services/router.py`)
- Instantiated as a module-level singleton via `get_llm_router()` so `CircuitBreaker` state persists across HTTP requests inside a worker process.
- `litellm.telemetry = False` and `litellm.suppress_debug_info = True` prevent background LiteLLM logging noise.
- **Exact Execution Cascade:**
  1. **Primary (`groq/llama-3.3-70b-versatile`):** `max_attempts=3`, `multiplier=0.5`, `max_wait=4.0`, `timeout=4.0`.
  2. **Secondary Fallback (`openai/gpt-4o-mini`):** Executed if breaker is `OPEN` or primary exhausts retries; wrapped in `execute_with_resilience(..., max_attempts=2, multiplier=0.5, max_wait=3.0)`.
  3. **Tertiary Fallback (`mistral/mistral-small-latest`):** Executed inside `_execute_fallback`'s `except` block as a single unretried `await litellm.acompletion(...)` call if secondary fails.
- Note how **retry amplification** works here: a single request hitting a failing primary + failing secondary triggers up to $3 + 2 + 1 = 6$ upstream attempts before raising—so keeping `max_attempts` tight (3 / 2 / 1) and `timeout=4.0s` bounds worst-case tail latency.

### D. Token Economics & Savings (`app/services/token_analytics.py`)
- **`MODEL_PRICING` (USD per 1M tokens):**
  - `groq/llama-3.3-70b-versatile`: `$0.59` prompt / `$0.79` completion
  - `openai/gpt-4o-mini`: `$0.15` prompt / `$0.60` completion
  - `openai/gpt-4o`: `$2.50` prompt / `$10.00` completion
  - `mistral/mistral-small-latest`: `$0.20` prompt / `$0.60` completion
- **`count_messages_tokens`:** Adds `+3` framing tokens per message (`<|start|>{role}\n{content}<|end|>`), `+1` if `"name"` is present, plus `+3` assistant priming tokens. Falls back to `cl100k_base` if `tiktoken.encoding_for_model(bare_model)` does not recognize a Groq/Mistral model name.
- **Cache Hit Savings:** `record_cache_hit_savings()` values hits against `baseline_model="openai/gpt-4o-mini"`, updates Redis via pipeline `INCRBYFLOAT` + `INCRBY`, and falls back to `_in_memory_*` counters if `redis_client is None` or Redis errors.

---

## 4. Practical Problems, Trade-Offs & Improvements
1. **Multi-Worker Circuit Breaker Drift:** `_global_router` stores `CircuitBreaker` in worker process RAM. With `--workers 4`, Worker 1 can trip `OPEN` after 5 failures while Workers 2–4 stay `CLOSED` until each hits 5 failures.
   - *Fix:* Store breaker state/timestamps in Redis Hash (`HSET circuit:groq ...`) + atomic Lua or Redis Pub/Sub invalidation.
2. **Special-Token Prompt Injection:** `tiktoken.encode(str(value))` in `token_analytics.py` uses default `allowed_special` behavior (which raises `ValueError` if user text literally contains `<|endoftext|>`). Passing `allowed_special=set()` / `disallowed_special=()` treats control token strings safely as plain text.
3. **SSE Streaming Failure Semantics:** Currently `chat.py` is buffered JSON, so failing over from Groq to OpenAI is invisible to clients (`200 OK`). Once SSE (`stream=True`) is enabled, once HTTP `200` headers and the first token chunk flush, you **cannot** change the HTTP status code or seamlessly switch models mid-sentence—you must emit an in-band SSE error event and close cleanly, and cancel the upstream generator on `asyncio.CancelledError` if the client disconnects so you stop paying for unconsumed tokens.

---

## 5. Quick Interview Questions
1. *Why pair Tenacity retries WITH a Circuit Breaker instead of using either one alone?*
2. *Explain why `asyncio.Lock` is required in `CircuitBreaker` and what happens if you use `threading.Lock`.*
3. *Why does `ResilientLLMRouter` wrap LiteLLM instead of LangChain, and how does it avoid retry storms?*
4. *How do you test a 3-retry + failover cascade deterministically in `pytest` without waiting 60 seconds or calling real APIs?*

