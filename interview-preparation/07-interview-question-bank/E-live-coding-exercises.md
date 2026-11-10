# Question Bank E: Production Live-Coding Exercises

Senior backend interviews rarely test only LeetCode puzzles—they frequently ask you to code the exact concurrency, caching, or resilience primitives used in your own project from scratch. Practice writing each of these 5 SentinelRAG building blocks cleanly in 10–15 minutes.

---

## Exercise 1: Async Cache-Aside Helper with TTL & LRU Eviction

**Prompt:** Implement an async, coroutine-safe in-memory cache helper with TTL expiration and LRU eviction at `max_capacity`.

```python
import asyncio
import time
from collections import OrderedDict
from collections.abc import Awaitable, Callable
from typing import TypeVar

T = TypeVar("T")

class AsyncTTLLRUCache:
    def __init__(self, max_capacity: int = 100) -> None:
        self.max_capacity = max_capacity
        self._data: OrderedDict[str, tuple[object, float]] = OrderedDict()
        self._lock = asyncio.Lock()

    async def get_or_compute(
        self, key: str, ttl_seconds: float, factory: Callable[[], Awaitable[T]]
    ) -> T:
        now = time.monotonic()
        async with self._lock:
            if key in self._data:
                val, expires_at = self._data[key]
                if now < expires_at:
                    self._data.move_to_end(key)
                    return val  # type: ignore[return-value]
                del self._data[key]

        # CRITICAL: Execute I/O outside self._lock so unrelated keys aren't blocked!
        computed = await factory()

        async with self._lock:
            self._data[key] = (computed, time.monotonic() + ttl_seconds)
            self._data.move_to_end(key)
            if len(self._data) > self.max_capacity:
                self._data.popitem(last=False)
        return computed
```
**Interview Talking Points:**
1. `time.monotonic()` never jumps backwards on NTP syncs (unlike `time.time()`).
2. Executing `await factory()` **outside** `self._lock` prevents a 1.5s LLM call on key A from stalling cache hits on key B.
3. Follow-up: *"What if 50 coroutines miss on the exact same key simultaneously?"* $\to$ Add per-key in-flight `asyncio.Future` coalescing (singleflight pattern).

---

## Exercise 2: Reciprocal Rank Fusion (RRF) from Scratch

**Prompt:** Given multiple ranked lists of document IDs (best at index 0), return deduplicated `(doc_id, score)` sorted descending using $k=60$.

```python
def reciprocal_rank_fusion(
    ranked_lists: list[list[str]], k: int = 60, top_k: int | None = None
) -> list[tuple[str, float]]:
    if k < 1:
        raise ValueError("k must be >= 1")
    scores: dict[str, float] = {}
    for doc_list in ranked_lists:
        # dict.fromkeys deduplicates accidental repeats within the same engine
        for rank, doc_id in enumerate(dict.fromkeys(doc_list), start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + (1.0 / (k + rank))

    ranked = sorted(scores.items(), key=lambda pair: (-pair[1], pair[0]))
    return ranked[:top_k] if top_k is not None else ranked
```
**Interview Talking Points:** `start=1` (1-based rank), `dict.fromkeys` intra-list dedup guard, $O(N \log N)$ complexity, and deterministic tie-breaking (`app/services/rrf.py` tie-breaks by `dense_score` then `sparse_score`).

---

## Exercise 3: Coroutine-Safe 3-State Circuit Breaker

**Prompt:** Build `CircuitBreaker` (`CLOSED`, `OPEN`, `HALF_OPEN`) with `asyncio.Lock` and single-canary protection in `HALF_OPEN`.

```python
import asyncio
import time
from enum import StrEnum

class CircuitState(StrEnum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: float = 60.0) -> None:
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = CircuitState.CLOSED
        self._failures = 0
        self._opened_at: float | None = None
        self._probe_in_flight = False
        self._lock = asyncio.Lock()

    async def can_execute(self) -> bool:
        async with self._lock:
            if self.state == CircuitState.CLOSED:
                return True
            if self.state == CircuitState.OPEN and self._opened_at is not None:
                if time.monotonic() - self._opened_at >= self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                    self._probe_in_flight = True
                    return True
                return False
            if self.state == CircuitState.HALF_OPEN and not self._probe_in_flight:
                self._probe_in_flight = True
                return True
            return False

    async def record_success(self) -> None:
        async with self._lock:
            self.state = CircuitState.CLOSED
            self._failures = 0
            self._opened_at = None
            self._probe_in_flight = False

    async def record_failure(self) -> None:
        async with self._lock:
            self._failures += 1
            self._probe_in_flight = False
            if self.state == CircuitState.HALF_OPEN or self._failures >= self.failure_threshold:
                self.state = CircuitState.OPEN
                self._opened_at = time.monotonic()
```
**Interview Talking Point:** Notice `_probe_in_flight` fixes the subtle multi-coroutine race where two callers arrive in `HALF_OPEN` before the first canary returns!

---

## Exercise 4: Exponential Backoff with Full Jitter

**Prompt:** Retry transient errors up to `max_attempts` using Full Jitter, while failing fast on permanent `4xx` errors.

```python
import asyncio
import random
from collections.abc import Awaitable, Callable
from typing import TypeVar

T = TypeVar("T")

class PermanentError(Exception): ...

async def retry_full_jitter(
    fn: Callable[[], Awaitable[T]],
    max_attempts: int = 3,
    base: float = 0.5,
    max_wait: float = 4.0,
) -> T:
    for attempt in range(max_attempts):
        try:
            return await fn()
        except PermanentError:
            raise
        except Exception:
            if attempt == max_attempts - 1:
                raise
            cap = min(max_wait, base * (2 ** attempt))
            await asyncio.sleep(random.uniform(0.0, cap))
    raise RuntimeError("unreachable")
```
**Interview Talking Point:** Full Jitter `Uniform(0, cap)` beats pure `base * 2^attempt` (which synchronizes herds) and Equal Jitter (`cap/2 + Uniform(0, cap/2)`).

---

## Exercise 5: Token Bucket Limiter (In-Memory Reference for Redis Lua)

**Prompt:** Implement lazy Token Bucket consumption returning `(allowed, retry_after_seconds)`.

```python
import asyncio
import math
import time

class TokenBucket:
    def __init__(self, capacity: int = 100, refill_per_sec: float = 10.0) -> None:
        self.capacity = float(capacity)
        self.refill_per_sec = float(refill_per_sec)
        self.tokens = float(capacity)
        self.last_updated = time.monotonic()
        self._lock = asyncio.Lock()

    async def consume(self, requested: int = 1) -> tuple[bool, int]:
        async with self._lock:
            now = time.monotonic()
            delta = max(0.0, now - self.last_updated)
            self.tokens = min(self.capacity, self.tokens + delta * self.refill_per_sec)
            self.last_updated = now
            if self.tokens >= requested:
                self.tokens -= requested
                return True, 0
            needed = requested - self.tokens
            return False, math.ceil(needed / self.refill_per_sec)
```
**Interview Talking Point:** In SentinelRAG (`app/core/rate_limiter.py`), this exact math is executed inside Redis Lua using `redis.call('TIME')` so `self._lock` is replaced by Redis's single-threaded atomicity across all container replicas.

---
**Next:** [08-scenario-based-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/08-scenario-based-questions.md)

