# Implementation Deep Dive 4: Circuit Breaker, Tenacity Router, Token Analytics, Chat Orchestrator & Dashboard

**Covered Modules (Verified in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md)):**
`app/core/circuit_breaker.py` | `app/services/resilience.py` | `app/services/token_analytics.py` | `app/services/router.py` | `app/api/v1/chat.py` | `app/schemas/chat.py` | `dashboard/app.py`

---

## 1. Coroutine-Safe Circuit Breaker (`app/core/circuit_breaker.py`)

### State Machine & Concurrency
- `CircuitState(StrEnum)`: `"CLOSED"`, `"OPEN"`, `"HALF_OPEN"`.
- Protected by `self._lock = asyncio.Lock()` across `can_execute()`, `record_success()`, `record_failure()`, `reset()`, and `trip()`.
- **Defaults (from `settings`):** `failure_threshold = 5`, `recovery_timeout = 60.0s`.
- **State Transitions:**
  - `can_execute()`: `CLOSED` $\to$ `True`; `OPEN` $\to$ checks `time.time() - _last_failure_time >= recovery_timeout`, transitions to `HALF_OPEN` and returns `True`, else `False`; `HALF_OPEN` $\to$ returns `True`.
  - `record_success()`: If `HALF_OPEN`, transitions to `CLOSED`; resets `_consecutive_failures = 0` and `_last_failure_time = None`.
  - `record_failure()`: Increments `_consecutive_failures`, stamps `_last_failure_time`. If `HALF_OPEN`, immediately re-trips to `OPEN`; if `CLOSED` and `_consecutive_failures >= failure_threshold`, trips to `OPEN`.
- **Subtle Production Improvement:** Adding an explicit `_probe_in_flight: bool` flag prevents multiple concurrent coroutines from all slipping through `HALF_OPEN` simultaneously before the first canary resolves, and replacing `time.time()` with `time.monotonic()` prevents NTP clock adjustments from distorting `recovery_timeout`.

---

## 2. Tenacity Resilience Engine (`app/services/resilience.py`)

- `is_transient_error(exc)`: Returns `True` for `RateLimitError`, `ServiceUnavailableError`, `InternalServerError`, `APIConnectionError`, `Timeout` (LiteLLM), `TimeoutError`, `ConnectionError`, `httpx.TimeoutException`, `httpx.NetworkError`, `status_code in (429, 502, 503, 504)`, or transient text indicators (`"rate limit"`, `"503"`, `"429"`, `"timed out"`, etc.). Permanent client errors (`400`, `401`, `403`, `404`) fail fast with zero retries.
- `create_resilient_retryer`: Configures `AsyncRetrying` with `stop_after_attempt(max_attempts)`, `wait_random_exponential(multiplier=multiplier, max=max_wait)` (Full Jitter), `retry_if_exception(is_transient_error)`, structured warning `before_sleep` hook, and `reraise=True`.

---

## 3. Multi-Provider Router (`app/services/router.py`)

### Singleton Pattern (`get_llm_router()`)
Returns module-level `_global_router` so `CircuitBreaker` state persists across incoming HTTP requests within a worker process (wiring `redis_client` onto `analytics` if initially `None`).

### Complete Failover Cascade (`ResilientLLMRouter.complete`)
1. **Circuit Check:** Calls `can_probe = await self.circuit_breaker.can_execute()`.
   - If `not can_probe or current_state == CircuitState.OPEN`: Logs `circuit_open_failing_fast_to_fallback` and jumps directly to `_execute_fallback` (`0ms` primary delay).
2. **Primary Invocation (`groq/llama-3.3-70b-versatile`):**
   - Calls `execute_with_resilience(litellm.acompletion, ..., timeout=4.0, max_attempts=3, multiplier=0.5, max_wait=4.0)`.
   - On success: `await circuit_breaker.record_success()`, extracts content + tokens, records spend in Redis, returns `RouterExecutionResult(fallback_triggered=False)`.
   - On exhaustion/error: `await circuit_breaker.record_failure()`, logs error, calls `_execute_fallback`.
3. **Secondary & Tertiary Fallback (`_execute_fallback`):**
   - Tries `self.fallback_model` (`openai/gpt-4o-mini`) via `execute_with_resilience(..., max_attempts=2, multiplier=0.5, max_wait=3.0)`.
   - If secondary *also* raises, logs `fallback_provider_failed_trying_tertiary` and issues one direct `await litellm.acompletion(model=self.tertiary_model, ...)` (`mistral/mistral-small-latest`).
   - Returns `RouterExecutionResult(fallback_triggered=True, fallback_reason=...)`.

---

## 4. Token & Cost Analytics (`app/services/token_analytics.py`)

- `get_tokenizer_for_model(model_name)`: Strips provider prefix (`"groq/llama-3.3-70b-versatile"` $\to$ `"llama-3.3-70b-versatile"`), calls `tiktoken.encoding_for_model()`, and gracefully falls back to `cl100k_base`.
- `count_messages_tokens`: Loops messages adding `+3` framing tokens each (`+1` for `"name"`) plus `+3` reply primer.
- `TokenAnalyticsTracker`: Uses atomic Redis `pipeline()` (`INCRBY` for token counts/hits/requests, `INCRBYFLOAT` for `dollars_spent` and `dollars_saved`) for both `analytics:tenant:{id}:*` and `analytics:global:*`, falling back to `_in_memory_*` floats/ints if Redis is `None` or fails.

---

## 5. Unified Chat Orchestrator (`app/api/v1/chat.py`) & Dashboard (`dashboard/app.py`)

### Orchestration in `chat_completions(...)`
- **Stage 1 (Cache Hit):** Checks `cache_service.get_or_lookup(...)` using last `user` message. On hit: increments `CACHE_OPERATIONS_TOTAL(result="hit")`, counts saved tokens against `gpt-4o-mini` baseline, records Redis + Prometheus `COST_SAVED_USD_TOTAL`, and returns `ChatCompletionResponse(model="sentinel-semantic-cache", telemetry=ChatTelemetry(cache_hit=True, provider="semantic_cache", cost_usd=0.0, ...))`.
- **Stage 2 (Hybrid Search):** If `use_retrieval=True`: executes `SearchRepository.search_hybrid_candidates(..., dense_limit=top_k_chunks*2, sparse_limit=top_k_chunks*2)`, fuses via `fuse_hybrid_candidates(..., k=60, top_k=payload.top_k_chunks)`, formats `[1] Source: {doc}\n{text}`, and inserts a `system` message at `augmented_messages[0]`. Wrapped in `try/except` to degrade gracefully to unaugmented LLM if DB fails.
- **Stage 3 (Router):** Calls `await router_service.complete(...)`.
- **Stage 4 (Cache Write & Metrics):** Writes response to Redis (`cache_service.set`), increments `CACHE_OPERATIONS_TOTAL(result="miss")` and `TOKENS_CONSUMED_TOTAL`, fetches `cumulative_saved`, and returns OpenAI-compatible `ChatCompletionResponse`.

### Streamlit Operational Dashboard (`dashboard/app.py`)
- Calls `POST {gateway_url}/api/v1/chat/completions` with configurable `tenant_id`, `X-API-Key`, `use_cache`, `use_retrieval`, `top_k_chunks`, and `selected_model`.
- Displays KPIs (`Cumulative Dollars Saved`, `Cache Hit Rate`, `Active Upstream Model`), cache-hit/fallback badges, and stage latency bars.
- *Important Honesty Note on Dashboard Code:* If the gateway API is unreachable, `dashboard/app.py` catches the connection exception and displays a simulated mock fallback, and its right-hand chunk list (`sample_chunks`) and Stage-1/2 latency progress bars are synthetic visual approximations computed from `telemetry["latency_ms"]` rather than direct PostgreSQL queries! Knowing that prevents getting tripped up if someone asks how `dashboard/app.py` gets chunk RRF scores.

---
**Next:** [07-interview-question-bank/](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/A-project-overview-and-system-flow.md)

