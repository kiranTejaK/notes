# Question Bank D: Implementation-Level & Operations Questions

Focuses strictly on SentinelRAG's actual functions, database queries, caching serialization, circuit breaker mechanics, testing, and deployment behavior.

---

## Q1: Why does `SearchRepository.search_hybrid_candidates()` run `search_dense()` and `search_sparse()` sequentially instead of with `asyncio.gather()`?

### Layer 1 – Short Answer
> "A SQLAlchemy `AsyncSession` wraps a single `asyncpg` connection over one half-duplex TCP socket. Calling `asyncio.gather()` on the same session causes two coroutines to interleave PostgreSQL wire-protocol frames, crashing immediately with `asyncpg.InterfaceError: another operation is in progress`."

### Layer 2 & 3 – Implementation Details (`app/repositories/search.py`)
- Each query uses a fast native index (`HNSW` for `<=>` cosine distance and `GIN` for `tsvector`), executing in milliseconds sequentially without checking out two connections from `db_manager`'s 20-connection pool per HTTP request.
- If true parallelism were ever needed (e.g., complex multi-table joins), you would explicitly acquire two independent `AsyncSession`s from `async_session_factory`—doubling pool concurrency pressure per request.

### Layer 5 – Practical Code Pattern
```python
# Safe sequential reuse on one AsyncSession (app/repositories/search.py)
dense_candidates = await self.search_dense(tenant_id, query_vector, dense_limit)
sparse_candidates = await self.search_sparse(tenant_id, query_text, sparse_limit)
```

---

## Q2: Explain how `RedisSemanticCache` stores and searches embeddings, and why it uses Base64 + `struct.pack`.

### Layer 1 – Short Answer
> "Storing a 1536-dimensional float array as JSON wastes ~28–35 KB of text and heavy string-parsing CPU. `serialize_embedding` packs the 1536 floats into 6,144 raw IEEE 754 `float32` bytes via `struct.pack`, then Base64-encodes them to 8,192 ASCII characters so `redis.asyncio` (`decode_responses=True`) never fails UTF-8 decoding."

### Layer 2 & 3 – Implementation Details (`app/services/cache.py`)
1. **Stage 1 (`O(1)`):** `compute_query_hash(query)` normalizes `query.strip().lower()` and computes `SHA-256` (`cache:{tenant_id}:{hash}`).
2. **Stage 2 (Semantic Scan):** Calls `SMEMBERS cache:{tenant_id}:keys`, fetches all hashes in one RTT via `pipeline(transaction=False)`, unpacks floats via `struct.unpack`, computes cosine similarity, and accepts a hit if `1.0 - best_similarity <= 0.08`.
3. **Ghost-Key Cleanup:** Any key in the set whose Redis Hash TTL (`86400s`) already expired returns `{}` from `HGETALL`; `get_or_lookup` lazily removes those dead names via `await self.redis.srem(index_key, *expired_keys)`.
4. **Endianness Improvement:** Adding `<` (`struct.pack(f"<{len(vector)}f", ...)`) makes binary bytes explicit little-endian across heterogeneous ARM64/x86 containers.

---

## Q3: How does `fuse_hybrid_candidates()` in `app/services/rrf.py` work, including tie-breaking and edge cases?

### Layer 1 – Short Answer
> "`fuse_hybrid_candidates` maps dense and sparse candidates by `chunk.id`, sums `1.0 / (k + rank)` with `k=60` and 1-based ranks, preserves component metadata (`dense_rank`, `sparse_rank`, scores), and sorts descending by `(fused_score, dense_score, sparse_score)`."

### Layer 3 & 5 – Implementation Edge Cases
- **Disjoint Sets:** A chunk appearing in only one list gets `1 / (60 + rank)` and `None` for the missing list's rank/score; a chunk in both lists adds both reciprocal ranks.
- **Deterministic Tie-Breaking:** If Chunk A is (#1 dense, #2 sparse) and Chunk B is (#2 dense, #1 sparse), their `fused_score` ties; Python's tuple sort key `(r.fused_score, r.dense_score or -1.0, r.sparse_score or -1.0)` breaks the tie using `dense_score`, then `sparse_score`.
- **Memory Optimization:** `DenseCandidate`, `SparseCandidate`, and `FusedSearchResult` all declare `@dataclass(slots=True)`, eliminating per-instance `__dict__` overhead.

---

## Q4: Walk through `CircuitBreaker` (`app/core/circuit_breaker.py`) and how `ResilientLLMRouter` integrates it with Tenacity.

### Layer 1 – Short Answer
> "`CircuitBreaker` uses an `asyncio.Lock` to guard `CLOSED` $\to$ `OPEN` (after `5` consecutive failures) and `OPEN` $\to$ `HALF_OPEN` (after `60s` cooldown). `ResilientLLMRouter.complete()` checks `can_execute()` before calling Groq; if `OPEN`, it fails fast in `0ms` to OpenAI `gpt-4o-mini` (with Mistral as tertiary fallback)."

### Layer 2 – Exact Control Flow (`app/services/router.py`, `app/services/resilience.py`)
1. `can_probe = await self.circuit_breaker.can_execute()`.
2. If `not can_probe or current_state == CircuitState.OPEN`: jump straight to `_execute_fallback(...)` (`0ms` primary delay).
3. Otherwise, run `execute_with_resilience(litellm.acompletion, model="groq/llama-3.3-70b-versatile", timeout=4.0, max_attempts=3, multiplier=0.5, max_wait=4.0)`.
   - `is_transient_error` retries only `429/502/503/504`/timeouts/network errors—never `400/401/403`.
4. If primary succeeds: `await circuit_breaker.record_success()` (resets failures to `0`, closes `HALF_OPEN` $\to$ `CLOSED`).
5. If primary raises after retries: `await circuit_breaker.record_failure()` (increments failures; trips to `OPEN` at `5`, or immediately re-trips if `HALF_OPEN`), then executes `_execute_fallback`:
   - Tries `openai/gpt-4o-mini` (`max_attempts=2`, `max_wait=3.0`).
   - If OpenAI also fails, falls back once to `mistral/mistral-small-latest`.

### Layer 4 – Follow-up Questions
- *What happens if 20 concurrent requests arrive right when the 60s `recovery_timeout` expires?*
  - In the current code, the first coroutine transitions `OPEN` $\to$ `HALF_OPEN` and returns `True`. Because `can_execute()` returns `self._state == CircuitState.HALF_OPEN` without a `_probe_in_flight` flag, subsequent coroutines arriving *before* the first probe finishes also enter `HALF_OPEN`! Adding a `_probe_in_flight: bool` boolean inside `self._lock` restricts `HALF_OPEN` strictly to one canary request.

---

## Q5: How does SentinelRAG manage SQLAlchemy session lifetimes, commits, and connection pooling?

### Layer 1 – Short Answer
> "A single `AsyncEngine` (`pool_size=20`, `max_overflow=10`, `pool_recycle=300`, `pool_pre_ping=True`) is paired with `async_session_factory(expire_on_commit=False)`. FastAPI's `get_async_session` generator yields an `AsyncSession`, rolls back on unhandled exceptions, and always closes in `finally`."

### Layer 3 – Key Nuances (`app/db/session.py` & `app/api/v1/documents.py`)
- **`expire_on_commit=False`:** Prevents SQLAlchemy from expiring ORM attributes after `await session.commit()`—because accessing expired attributes asynchronously outside a greenlet raises `MissingGreenlet`.
- **Explicit Commit:** `get_async_session` does **not** auto-commit on exit, so write handlers (`batch_ingest_documents`) explicitly call `await session.commit()` after `await session.scalars(insert(DocumentChunk).returning(DocumentChunk.id), chunk_records)`.
- **`pool_pre_ping=True`:** Executes `SELECT 1` on connection checkout to transparently discard half-closed TCP sockets before running user queries.

---

## Q6: How is SentinelRAG tested asynchronously without calling paid LLMs or spinning up live network ports?

### Layer 1 – Short Answer
> "The 44 tests across 12 modules run via `pytest-asyncio` (`asyncio_mode = 'auto'`) using `httpx.AsyncClient(transport=ASGITransport(app=app))` for in-memory ASGI dispatch, `app.dependency_overrides` to inject `AsyncMock` sessions and `FakeRedis`, and `unittest.mock.patch` on `litellm.acompletion`."

### Layer 2 – Key Testing Patterns (`tests/*.py`)
- **In-Memory Simulators:** `FakeRedis` (`test_semantic_cache.py`) simulates hashes, sets, TTLs, and pipelines; `FakeRateLimitRedis` (`test_rate_limiter.py`) simulates `simulated_time` advancement so replenishment tests run in milliseconds without `asyncio.sleep`.
- **Failure Sequence Mocking (`test_router.py`, `test_telemetry_and_e2e.py`):** Uses `mock_llm.side_effect = [ServiceUnavailableError(...), ..., fallback_resp]` to deterministically verify 3 Tenacity retries + Circuit Breaker trip + OpenAI failover.
- **Teardown Safety:** Always clears `app.dependency_overrides.clear()` inside `try ... finally:` so overrides never bleed across test files.
- **Known Testing Gap:** Because all 44 tests mock Postgres/Redis/LiteLLM, adding a CI integration job against the real `pgvector` and `redis` Docker service containers (running actual HNSW/GIN SQL and real Lua scripts without mocks) is a high-value improvement.

---
**Next:** [E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md)

