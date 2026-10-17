# SentinelRAG: Testing, Debugging & Failure Scenarios

> **Target Audience:** Python Backend Developers (2–4 Years Experience) preparing for Testing, Code Quality & Production Troubleshooting Rounds.  
> **Philosophy:** Senior engineers don't just write happy-path tests; they write tests that prove systems fail gracefully under network partitions, rate limits, timeouts, and corrupted data.  
> **Workspace Root:** [c:/Users/kiran/Desktop/sentinel-rag](file:///c:/Users/kiran/Desktop/sentinel-rag)

---

## 1. Analysis of Existing Test Suite

SentinelRAG includes **44 automated test suites** organized under the [tests/](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/) directory. All tests run via `pytest` with `pytest-asyncio` in auto mode (`asyncio_mode = "auto"` in [pyproject.toml#L62-L66](file:///c:/Users/kiran/Desktop/sentinel-rag/pyproject.toml#L62-L66)).

### Inventory of Existing Test Modules

| Test File | Primary Scope & Tested Unit | Techniques Used |
| :--- | :--- | :--- |
| [tests/test_chat_completions.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_chat_completions.py) | Full RAG pipeline: cache hit, cache miss, hybrid retrieval, and provider fallback. | `app.dependency_overrides`, `patch` on `SearchRepository` and `litellm.acompletion`, `httpx.AsyncClient`. |
| [tests/test_circuit_breaker.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_circuit_breaker.py) | Circuit Breaker state transitions (`CLOSED` $\to$ `OPEN` $\to$ `HALF_OPEN` $\to$ `CLOSED`). | Monotonic clock simulation, simulated consecutive failures, state assertion. |
| [tests/test_document_ingestion.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_document_ingestion.py) | Bulk chunk insertion, vector validation, and generated `tsvector` insertion. | `AsyncMock` session override, Pydantic schema validation tests. |
| [tests/test_healthz.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_healthz.py) | Subsystem liveness and readiness ping checks (HTTP 200 vs 503). | Mocked `db_manager.ping()` and `redis_manager.ping()` simulating database drops. |
| [tests/test_hybrid_search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_hybrid_search.py) | Dual-retrieval candidate querying and RRF merging. | Mocked repository candidate lists, vector length validation (1536d). |
| [tests/test_rate_limiter.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_rate_limiter.py) | Token Bucket Lua script execution, burst handling, and HTTP 429 response formatting. | Mocked Redis script evaluation, header inspection (`X-RateLimit-Limit`, `Retry-After`). |
| [tests/test_resilience.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_resilience.py) | Tenacity retry engine with Full Jitter and transient exception classification. | Raising mock transient exceptions (`RateLimitError`, `503`), verifying retry counts. |
| [tests/test_router.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_router.py) | Groq $\to$ OpenAI $\to$ Mistral failover cascade. | Mocking upstream LiteLLM responses, inspecting fallback triggers. |
| [tests/test_rrf.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_rrf.py) | Pure mathematical Reciprocal Rank Fusion unit tests ($k=60$). | Edge cases: disjoint rankings, empty lists, tie-breaking logic. |
| [tests/test_semantic_cache.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_semantic_cache.py) | Exact SHA-256 match, vector cosine similarity distance evaluation, and TTL expiration. | Base64 packing verification, synthetic vector similarity math, mock Redis pipelines. |
| [tests/test_telemetry_and_e2e.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_telemetry_and_e2e.py) | Correlation ID propagation, `structlog` ContextVars, Prometheus counter incrementing. | Inspecting `X-Request-Id` headers and Prometheus exposition at `/metrics`. |
| [tests/test_token_analytics.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_token_analytics.py) | `tiktoken` BPE token counting, message formatting overhead, and dollar calculations. | Comparing known strings to exact BPE token numbers across model pricing tiers. |

---

## 2. Testing Methodologies in SentinelRAG

### A. Testing Async Route Handlers via `httpx.AsyncClient` & `ASGITransport`
In FastAPI applications, testing asynchronous routes without spinning up a live network port is achieved using `httpx.ASGITransport`:

```python
# Standard Pattern used across SentinelRAG tests
import pytest
from httpx import ASGITransport, AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_endpoint():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get("/healthz")
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"
```
**Why this matters:** `ASGITransport` invokes FastAPI's ASGI interface directly in-memory within the test event loop. It avoids opening TCP ports, preventing port conflict issues during parallel CI runs.

---

### B. Dependency Overrides vs. `unittest.mock.patch`
SentinelRAG uses FastAPI's `app.dependency_overrides` for stateful dependencies (database sessions, Redis connections) and `patch` for internal service methods:

```python
# Code Pattern: tests/test_hybrid_search.py
mock_session = AsyncMock()

async def override_get_async_session():
    yield mock_session

# Inject mock into FastAPI's DI container
app.dependency_overrides[get_async_session] = override_get_async_session

try:
    # Run tests against client...
    ...
finally:
    # CRITICAL: Always clear overrides to prevent state contamination between test files!
    app.dependency_overrides.clear()
```

---

## 3. Production Failure Scenarios & Debugging Playbook

### Failure Scenario 1: `asyncpg.exceptions.InterfaceError: cannot perform operation: another operation is in progress`

#### Symptoms & Root Cause
- **Symptom:** API routes executing concurrent database queries suddenly crash with HTTP 500. Logs display `InterfaceError: cannot perform operation: another operation is in progress`.
- **Root Cause:** A developer refactored a route to use `asyncio.gather(query_1, query_2)` using the *same* `AsyncSession`. `asyncpg` operates a single half-duplex socket per connection. When two coroutines attempt to read/write concurrently on that socket, the driver asserts an `InterfaceError`.
- **Debugging & Verification:**
  Check the stack trace for `asyncio.gather` or concurrent coroutines wrapping `session.execute()`.
- **Resolution:**
  Run queries sequentially (as done in [SearchRepository.search_hybrid_candidates](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L140-L151)), or check out two distinct sessions from the connection pool if concurrent execution is mandatory.

---

### Failure Scenario 2: Redis Connection Drop / Socket Timeout Under High Load

#### Symptoms & Root Cause
- **Symptom:** Gateway begins throwing timeout warnings: `rate_limit_redis_evaluation_error` or `semantic_cache_lookup_failed`.
- **Root Cause:** Redis server reached `maxclients`, or a slow blocking command (e.g., an unindexed `KEYS *` instead of `SCAN`) blocked the Redis single-threaded event loop, exceeding SentinelRAG's `REDIS_TIMEOUT_MS = 500`.
- **SentinelRAG Defensive Safeguard:**
  In [app/core/rate_limiter.py#L161-L168](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L161-L168), the rate limiter implements a **Fail-Open Policy**:
  ```python
  except Exception as exc:
      logger.error("rate_limit_redis_evaluation_error", error=str(exc))
      # Fail-open policy maintains gateway availability during Redis hiccups
      return RateLimitResult(allowed=True, remaining=capacity, retry_after=0, limit=capacity)
  ```
  Similarly, [app/services/cache.py#L259-L271](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L259-L271) treats cache connection errors as a transparent `CACHE_MISS`, allowing the request to proceed to downstream retrieval rather than crashing.

---

### Failure Scenario 3: Database Connection Pool Exhaustion (`QueuePool limit of size 20 overflow 10 reached`)

#### Symptoms & Root Cause
- **Symptom:** API latency spikes from 100ms to 30.0s. Requests fail with `TimeoutError: QueuePool limit of size 20 overflow 10 reached, connection timed out, timeout 30.00`.
- **Root Cause:** Connections checked out from `async_session_factory` are not being closed, or a long-running transaction is holding a connection open while awaiting an external network call.
- **Debugging Playbook:**
  1. Check [app/db/session.py#L41-L56](file:///c:/Users/kiran/Desktop/sentinel-rag/app/db/session.py#L41-L56). Verify that `yield session` is wrapped in `try...finally: await session.close()`.
  2. Inspect routes: **Never** hold an open database transaction while awaiting an external LLM call (`litellm.acompletion`). Complete the database query, commit/close the session, and *then* call the external API.
  3. Query PostgreSQL active connections:
     ```sql
     SELECT pid, age(clock_timestamp(), query_start), usename, query, state
     FROM pg_stat_activity
     WHERE state != 'idle' AND query NOT ILIKE '%pg_stat_activity%'
     ORDER BY query_start ASC;
     ```

---

### Failure Scenario 4: "Circuit Breaker Drift" Across Multi-Worker Pods

#### Symptoms & Root Cause
- **Symptom:** Groq is down, but some requests still experience 4-second timeouts while others fail-fast in 0ms.
- **Root Cause:** Uvicorn is running with multiple worker processes (`uvicorn app.main:app --workers 4`). The current `CircuitBreaker` stores state in Python memory (`self._state`). Worker 1 might have recorded 5 failures and tripped to `OPEN`, but Worker 2 has only handled 2 failures and remains `CLOSED`.
- **How to Explain This in an Interview:**
  > *"Our current Circuit Breaker is coroutine-safe in local memory per worker process. Under multi-worker deployments, this causes minor drift where each worker evaluates the failure threshold independently. In our production scaling roadmap, we document externalizing circuit state to a Redis Hash (`HSET circuit:groq state OPEN`) and using Redis Pub/Sub to broadcast state transitions instantly across all Uvicorn worker pods."*

---

## 4. Concrete Recommended Tests to Add (Gaps Identified)

While the existing 44 tests provide high coverage, an interview candidate should propose these three production hardening tests:

### Recommended Test 1: Concurrency Race Condition Test on Rate Limiter
Verify that 50 concurrent requests hitting the rate limiter simultaneously do not exceed capacity.

```python
# Recommended Test: tests/test_rate_limiter_concurrency.py
import asyncio
import pytest
from app.core.rate_limiter import TokenBucketLimiter

@pytest.mark.asyncio
async def test_concurrent_burst_does_not_exceed_capacity():
    # Capacity = 10 tokens, refill = 1 token/sec
    limiter = TokenBucketLimiter()
    # Mock redis client with atomic script evaluation...
    tasks = [
        limiter.check(identifier="test_user", capacity=10, refill_rate_per_sec=1.0)
        for _ in range(25)
    ]
    results = await asyncio.gather(*tasks)
    allowed_count = sum(1 for r in results if r.allowed)
    assert allowed_count == 10  # Exactly 10 allowed, 15 rejected!
```

---

### Recommended Test 2: Circuit Breaker Canary Race Condition Test
Verify that when the recovery timeout expires, only **one** coroutine enters `HALF_OPEN` while other concurrent requests wait or fail fast.

```python
# Recommended Test: tests/test_circuit_breaker_concurrency.py
import asyncio
import time
import pytest
from app.core.circuit_breaker import CircuitBreaker, CircuitState

@pytest.mark.asyncio
async def test_circuit_breaker_single_canary_probe():
    cb = CircuitBreaker(failure_threshold=2, recovery_timeout=0.1)
    await cb.record_failure()
    await cb.record_failure()
    assert cb.state == CircuitState.OPEN

    # Wait for recovery timeout
    await asyncio.sleep(0.15)

    # 10 coroutines simultaneously probe can_execute()
    probe_results = await asyncio.gather(*(cb.can_execute() for _ in range(10)))
    # With asyncio.Lock, exactly one probe transitions the breaker to HALF_OPEN
    assert cb.state == CircuitState.HALF_OPEN
```

---

### Recommended Test 3: Large Payload & Schema Boundary Injection Test
Verify that payloads with 100,000-character messages or invalid vector dimensions are rejected cleanly with HTTP 422 before allocating database connections.

```python
# Recommended Test: tests/test_schema_boundaries.py
import pytest
from httpx import ASGITransport, AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_invalid_vector_dimension_rejected_before_db():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # Pass 512d vector instead of required 1536d
        payload = {
            "query": "test query",
            "query_vector": [0.1] * 512,
            "tenant_id": "tenant_1"
        }
        res = await client.post("/api/v1/search/hybrid", json=payload)
        assert res.status_code == 422
        assert "1536" in str(res.json())
```
