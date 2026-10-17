# SentinelRAG: Code Walkthroughs & Targeted Coding Exercises

> **Target Audience:** Python Backend Developers (2–4 Years Experience) preparing for Live Coding & Technical Implementation Rounds.  
> **Philosophy:** No abstract DSA or toy LeetCode puzzles. Focus entirely on production backend engineering: asynchronous I/O, state machines, caching, resilience, rate limiting, and defensive concurrency.  
> **Workspace Root:** [c:/Users/kiran/Desktop/sentinel-rag](file:///c:/Users/kiran/Desktop/sentinel-rag)

---

## 1. Deep Code Walkthrough: Resilient LLM Router & Fallback

### Existing Implementation Walkthrough
- **File:** [app/services/router.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py)
- **Function:** `ResilientLLMRouter.complete(...)` ([lines 75–203](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py#L75-L203))
- **Inputs:**
  - `messages: list[dict[str, Any]]`: Formatted chat messages including system prompt, augmented context, and user query.
  - `tenant_id: str`: Multi-tenant identifier for tracking token spend.
  - `temperature: float`: Sampling temperature (default: 0.7).
  - `max_tokens: int`: Maximum completion length (default: 1024).
  - `preferred_model: str | None`: Optional model override (e.g., `openai/gpt-4o-mini`).
- **Outputs:**
  - `RouterExecutionResult`: Dataclass containing `content`, `provider`, `model_used`, `fallback_triggered`, `cost_usd`, `latency_ms`, `circuit_breaker_state`.

```python
# Code Walkthrough: app/services/router.py
async def complete(self, messages, tenant_id="default", ...):
    start_time = time.perf_counter()
    target_primary = preferred_model or self.primary_model

    # 1. Check Circuit Breaker State
    can_probe = await self.circuit_breaker.can_execute()
    current_state = self.circuit_breaker.state

    # 2. FAST PATH: Circuit OPEN -> Immediate Fail-Fast
    if not can_probe or current_state == CircuitState.OPEN:
        return await self._execute_fallback(
            messages=messages,
            tenant_id=tenant_id,
            fallback_model=self.fallback_model,
            fallback_reason="circuit_breaker_open_fail_fast",
            ...
        )

    # 3. NORMAL PATH: Try Primary Model with Exponential Backoff
    try:
        response = await execute_with_resilience(
            litellm.acompletion,
            model=target_primary,
            messages=messages,
            timeout=settings.CIRCUIT_BREAKER_TIMEOUT_SECONDS,
            max_attempts=3,
            ...
        )
        await self.circuit_breaker.record_success()
        ...
        return RouterExecutionResult(...)
    except Exception as primary_error:
        # 4. FAILURE PATH: Trip Breaker and Route to Fallback
        await self.circuit_breaker.record_failure()
        return await self._execute_fallback(...)
```

### Core Logic vs. Production Robustness Safeguards
An interviewer might ask: *"Why is this router 150 lines long when `litellm.acompletion` is just one line?"*

| Implementation Element | Minimal Implementation | SentinelRAG Production Safeguard | Why It Must NOT Be Removed |
| :--- | :--- | :--- | :--- |
| **Circuit State Probe** | Direct API call | `await self.circuit_breaker.can_execute()` | Without it, every request during an outage waits for the 4s timeout, freezing worker threads. |
| **Timeout Guard** | Default provider timeout (60s–120s) | Hard `timeout=4.0` in `execute_with_resilience` | Prevents hung TCP connections and socket starvation under load. |
| **Token & Spend Tracking** | Relies on provider dashboard | `self.analytics.record_llm_execution(...)` | Provides immediate internal chargeback attribution and prevents runaway billing. |
| **Tertiary Fallback** | Single fallback | Groq $\to$ OpenAI $\to$ Mistral cascade | If OpenAI also undergoes a regional outage, the service still degrades gracefully to Mistral. |

---

## 2. Interview Coding Exercise 1: Implement an In-Memory Cache-Aside Helper with TTL & LRU Eviction

### Interviewer Prompt
> *"Write a reusable, asynchronous Cache-Aside helper in Python. It should take an asynchronous factory function, a cache key, and a TTL in seconds. If the key exists and has not expired, return the cached value. Otherwise, call the factory function, store the result with its expiration timestamp, and return it. If the cache exceeds a maximum capacity (e.g., 100 items), evict the Least Recently Used (LRU) entry."*

### Solution & Detailed Explanation
```python
import time
import asyncio
from typing import Any, Callable, Coroutine
from collections import OrderedDict

class AsyncLRUCache:
    """Coroutine-safe in-memory cache-aside engine with TTL and LRU eviction."""

    def __init__(self, max_capacity: int = 100):
        self.max_capacity = max_capacity
        # OrderedDict maintains insertion/access order: key -> (value, expiry_timestamp)
        self._cache: OrderedDict[str, tuple[Any, float]] = OrderedDict()
        self._lock = asyncio.Lock()

    async def get_or_compute(
        self,
        key: str,
        ttl_seconds: float,
        factory_coro: Callable[[], Coroutine[Any, Any, Any]],
    ) -> Any:
        now = time.monotonic()

        async with self._lock:
            # 1. Check if key exists in cache
            if key in self._cache:
                value, expiry = self._cache[key]
                if now < expiry:
                    # Cache Hit: Move to end to mark as Most Recently Used
                    self._cache.move_to_end(key)
                    return value
                else:
                    # Expired entry: delete immediately
                    del self._cache[key]

        # 2. Cache Miss: Execute the factory coroutine outside the lock to avoid holding the lock during I/O!
        computed_value = await factory_coro()

        async with self._lock:
            # 3. Store in cache with future expiry
            expiry = time.monotonic() + ttl_seconds
            self._cache[key] = (computed_value, expiry)
            self._cache.move_to_end(key)

            # 4. Enforce max capacity (evict oldest / least recently used)
            if len(self._cache) > self.max_capacity:
                oldest_key, _ = self._cache.popitem(last=False)

        return computed_value
```

### Critical Line-by-Line Technical Analysis
1. `time.monotonic()` vs `time.time()`: Always use `time.monotonic()` for cache TTL and timers. `time.time()` uses wall-clock system time, which can jump backwards during NTP sync adjustments or daylight savings changes.
2. `OrderedDict.move_to_end(key)`: Gives $O(1)$ amortized LRU tracking. When a hit occurs, the key is moved to the right. `popitem(last=False)` pops from the left, evicting the least recently used item in $O(1)$.
3. **Lock Scope (Critical Concurrency Insight):** Notice `await factory_coro()` is executed **outside** the lock. If an engineer holds `async with self._lock:` while awaiting an external I/O call (e.g., an LLM query taking 1500ms), all other concurrent requests trying to read or write *any* key in the cache are blocked!

### Unit Test Case
```python
import pytest

@pytest.mark.asyncio
async def test_async_lru_cache():
    cache = AsyncLRUCache(max_capacity=2)
    computed_count = 0

    async def fetch_user():
        nonlocal computed_count
        computed_count += 1
        return {"user": "alice"}

    # First fetch: computes
    res1 = await cache.get_or_compute("user_1", ttl_seconds=10.0, factory_coro=fetch_user)
    assert res1 == {"user": "alice"}
    assert computed_count == 1

    # Second fetch: cache hit (factory not called)
    res2 = await cache.get_or_compute("user_1", ttl_seconds=10.0, factory_coro=fetch_user)
    assert res2 == {"user": "alice"}
    assert computed_count == 1
```

---

## 3. Interview Coding Exercise 2: Implement Reciprocal Rank Fusion (RRF) from Scratch

### Interviewer Prompt
> *"Given two ranked lists of document IDs (one from dense vector search and one from BM25 keyword search), implement the Cormack et al. Reciprocal Rank Fusion algorithm. Each list contains document IDs sorted from best (rank 1) to worst. Return a deduplicated list of document IDs sorted by their fused score in descending order. Use smoothing parameter $k=60$."*

### Solution & Detailed Explanation
```python
def reciprocal_rank_fusion(
    rankings: list[list[str]],
    k: int = 60,
    top_k: int | None = None,
) -> list[tuple[str, float]]:
    """Calculates Cormack et al. (2009) Reciprocal Rank Fusion across ranked candidate lists."""
    if k < 1:
        raise ValueError(f"Smoothing constant k must be >= 1, got {k}")

    fused_scores: dict[str, float] = {}

    # Iterate over each ranked modality list
    for ranking in rankings:
        # 1-based rank indexing
        for rank_idx, doc_id in enumerate(ranking, start=1):
            # Mathematical RRF formula: sum(1 / (k + rank))
            fused_scores[doc_id] = fused_scores.get(doc_id, 0.0) + (1.0 / (k + rank_idx))

    # Sort descending by fused score
    sorted_candidates = sorted(
        fused_scores.items(),
        key=lambda item: item[1],
        reverse=True,
    )

    if top_k is not None:
        sorted_candidates = sorted_candidates[:top_k]

    return sorted_candidates
```

### Edge Cases to Discuss with Interviewer
1. **Disjoint Result Sets:** What if a document appears only in the dense search list and not in the sparse list? RRF naturally handles this; it gets $\frac{1}{60 + r_{\text{dense}}} + 0$.
2. **Duplicate IDs within the same list:** If an upstream query engine accidentally returns the same document ID twice in one list, `enumerate` will count both. Production code should pass `ranking = list(dict.fromkeys(ranking))` to deduplicate ranks within a single modality.
3. **Empty Input Lists:** If sparse search finds 0 matches for an obscure keyword, `rankings` contains `[dense_list, []]`. RRF gracefully completes with dense candidates only.

---

## 4. Interview Coding Exercise 3: Implement a State-Machine Circuit Breaker

### Interviewer Prompt
> *"Implement a coroutine-safe Circuit Breaker class with CLOSED, OPEN, and HALF_OPEN states. When in CLOSED, count consecutive failures; if failures reach 3, trip to OPEN. In OPEN, fail fast immediately for 10 seconds. After 10 seconds, allow a single canary probe in HALF_OPEN. If the probe succeeds, recover to CLOSED. If it fails, trip back to OPEN for another 10 seconds."*

### Solution & Detailed Explanation
```python
import asyncio
import time
from enum import Enum

class CircuitState(str, Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreakerOpenError(Exception):
    pass

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 3, recovery_timeout: float = 10.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = CircuitState.CLOSED
        self.consecutive_failures = 0
        self.last_failure_time: float | None = None
        self._lock = asyncio.Lock()

    async def can_execute(self) -> bool:
        async with self._lock:
            if self.state == CircuitState.CLOSED:
                return True

            if self.state == CircuitState.OPEN:
                now = time.monotonic()
                if self.last_failure_time is not None and (now - self.last_failure_time >= self.recovery_timeout):
                    # Cooldown elapsed -> transition to HALF_OPEN for canary probe
                    self.state = CircuitState.HALF_OPEN
                    return True
                return False

            # In HALF_OPEN: only one canary is allowed
            return self.state == CircuitState.HALF_OPEN

    async def record_success(self) -> None:
        async with self._lock:
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED
            self.consecutive_failures = 0
            self.last_failure_time = None

    async def record_failure(self) -> None:
        async with self._lock:
            self.consecutive_failures += 1
            self.last_failure_time = time.monotonic()

            if self.state == CircuitState.HALF_OPEN or self.consecutive_failures >= self.failure_threshold:
                self.state = CircuitState.OPEN

    async def call(self, coro_func, *args, **kwargs):
        """Wrapper method executing a protected coroutine."""
        if not await self.can_execute():
            raise CircuitBreakerOpenError("Circuit is OPEN. Fail fast.")

        try:
            result = await coro_func(*args, **kwargs)
            await self.record_success()
            return result
        except Exception:
            await self.record_failure()
            raise
```

---

## 5. Interview Coding Exercise 4: Exponential Backoff with Full Jitter

### Interviewer Prompt
> *"Write an asynchronous retry decorator or function in Python that retries an async callable up to 3 times on transient exceptions (like HTTP 429 or 503). Use Exponential Backoff with Full Jitter. Do not retry on client 400 errors."*

### Solution & Detailed Explanation
```python
import asyncio
import random
from typing import Any, Callable, Coroutine

class PermanentClientError(Exception):
    """Non-retryable 4xx client errors."""
    pass

class TransientServiceError(Exception):
    """Retryable 5xx or 429 rate limit errors."""
    pass

async def retry_with_full_jitter(
    coro_fn: Callable[..., Coroutine[Any, Any, Any]],
    *args: Any,
    max_attempts: int = 3,
    base_delay: float = 0.5,
    max_delay: float = 4.0,
    **kwargs: Any,
) -> Any:
    """Executes a coroutine with Exponential Backoff and Full Jitter."""
    for attempt in range(1, max_attempts + 1):
        try:
            return await coro_fn(*args, **kwargs)
        except PermanentClientError:
            # Fail fast on non-retryable errors
            raise
        except Exception as exc:
            if attempt == max_attempts:
                # Retries exhausted
                raise

            # Full Jitter formula: T_sleep ~ Uniform(0, min(max_delay, base * 2^(attempt - 1)))
            exponential_cap = min(max_delay, base_delay * (2 ** (attempt - 1)))
            sleep_time = random.uniform(0.0, exponential_cap)

            print(f"[Attempt {attempt} Failed: {exc}] Retrying in {sleep_time:.3f}s with Full Jitter...")
            await asyncio.sleep(sleep_time)
```

---

## 6. Interview Coding Exercise 5: Token Bucket Rate Limiter with Monotonic Replenishment

### Interviewer Prompt
> *"Write an in-memory Token Bucket rate limiter class in Python. It should take capacity and refill_rate_per_sec. It must support consuming 1 token. If tokens are available, return True; otherwise return False along with the retry_after duration in seconds."*

### Solution & Detailed Explanation
```python
import time
import math
import asyncio

class TokenBucket:
    def __init__(self, capacity: int = 10, refill_rate_per_sec: float = 1.0):
        self.capacity = float(capacity)
        self.refill_rate = float(refill_rate_per_sec)
        self.tokens = float(capacity)
        self.last_updated = time.monotonic()
        self._lock = asyncio.Lock()

    async def consume(self, requested: int = 1) -> tuple[bool, int]:
        """Atomically evaluates token bucket. Returns (allowed, retry_after_seconds)."""
        async with self._lock:
            now = time.monotonic()
            elapsed = max(0.0, now - self.last_updated)
            self.last_updated = now

            # Replenish tokens based on elapsed seconds
            self.tokens = min(self.capacity, self.tokens + (elapsed * self.refill_rate))

            if self.tokens >= requested:
                self.tokens -= requested
                return True, 0
            else:
                # Calculate required wait time for needed tokens
                needed = requested - self.tokens
                retry_after = math.ceil(needed / self.refill_rate)
                return False, retry_after
```

### Integration with SentinelRAG
In SentinelRAG, this exact algorithm was ported to Lua ([TOKEN_BUCKET_LUA_SCRIPT](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L17-L69)) so that `self._lock` is replaced by Redis's single-threaded event loop, making the bucket cluster-safe across stateless Docker containers.
