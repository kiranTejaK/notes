# SentinelRAG — Prioritized Learning Roadmap

> **Document Purpose:** Practical, actionable preparation checklist and progress tracker for mastering the SentinelRAG AI Middleware Gateway.
> This roadmap guides you through a structured 5-phase learning sequence designed to maximize interview retention while keeping cognitive load manageable.
>
> **Companion Documents:**
> - [`01_SentinelRAG_Project_Walkthrough.md`](./01_SentinelRAG_Project_Walkthrough.md) — Comprehensive source-code walkthrough (IDE companion).
> - [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md) — Three-level prioritized technical concepts (22 Essential, 14 Supporting, 32 Supplementary).
> - [`03_SentinelRAG_Interview_QA.md`](./03_SentinelRAG_Interview_QA.md) — Prioritized question bank (75 Essential, 130 Supplementary reference questions).

---

## Preparation Overview & Self-Paced Strategy

Preparing for a high-concurrency AI gateway project requires balancing **system design breadth**, **underlying CS depth**, and **hands-on code fluency**. Instead of attempting to memorize all 68 concepts and 205 questions at once, follow this prioritized 5-phase pipeline:

```
+----------------------------------------------------------------------------------------------------+
| Phase 1: Core Concepts — First Pass   | Master the 22 Category A concepts (mental model).          |
+---------------------------------------+------------------------------------------------------------+
| Phase 2: Project Walkthrough          | Step through source files in IDE alongside Doc 01.         |
+---------------------------------------+------------------------------------------------------------+
| Phase 3: Core Concepts — Second Pass  | Deepen technical rigor; study Category B supporting topics. |
+---------------------------------------+------------------------------------------------------------+
| Phase 4: Interview Q&A Practice       | Master the 75 Category A interview questions out loud.     |
+---------------------------------------+------------------------------------------------------------+
| Phase 5: Practical Code Familiarity   | Validate edge cases, inspect tests, and explain live code.  |
+----------------------------------------------------------------------------------------------------+
```

---

## Phase 1: Core Concepts — First Pass (Mental Model Foundation)

**Objective:** Build a rock-solid architectural vocabulary before touching lines of code. Do not worry about implementation edge cases or supplementary topics yet—focus strictly on the **22 Category A Essential Concepts** in [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md).

### Phase 1 Action Checklist
- [ ] Read **Priority Learning Summary** in [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md).
- [ ] **A1 — ASGI Specification vs. WSGI:** Understand why WSGI blocks on streaming/SSE and how ASGI `scope/receive/send` handles concurrent I/O.
- [ ] **A2 — FastAPI Dependency Injection (`Depends`):** Understand request-scoped dependencies, `yield` generator cleanup, and `AsyncExitStack`.
- [ ] **A3 — `BackgroundTasks`:** Understand how post-response fire-and-forget hooks eliminate Redis cache write latency, and know the durability trade-off.
- [ ] **A4 — Middleware Execution Order:** Master Starlette's onion wrapping order: why the *last registered* middleware is the *outermost* wrapper.
- [ ] **A5 — `StreamingResponse` & Server-Sent Events (SSE):** Understand `text/event-stream` wire framing, `X-Accel-Buffering: no`, and client disconnection cancellation.
- [ ] **A6 — Event Loop and Coroutines:** Understand single-threaded cooperative multitasking and non-blocking socket multiplexing.
- [ ] **A7 — Context-Local State via `contextvars`:** Understand how `structlog.contextvars` isolates `request_id` and `tenant_id` across concurrent asyncio tasks.
- [ ] **A8 — `asyncpg` & SQLAlchemy 2.0 Async:** Understand native PostgreSQL async binary protocol and connection pool sizing (`DB_POOL_SIZE=20`, `DB_MAX_OVERFLOW=10`).
- [ ] **A9 — `pgvector` Extension & Vector Indexing:** Understand dense vector columns, `<=>` cosine distance, and `vector_cosine_ops` HNSW indexing.
- [ ] **A10 — PostgreSQL Full-Text Search (`TSVector` / `TSQuery`):** Understand generated persisted `tsvector` columns, GIN indexes, and `ts_rank_cd` scoring.
- [ ] **A11 — Redis Pipelines & Command Batching:** Understand how pipelining eliminates round-trip latency (`transaction=True` vs `transaction=False`).
- [ ] **A12 — Atomic Lua Scripts in Redis:** Understand atomic read-modify-write execution on the Redis server without distributed locking overhead.
- [ ] **A13 — Redis Stack / RediSearch:** Understand `FT.CREATE` HNSW vector indexes and why `@tenant_id:{...}` TAG pre-filtering is vital for tenant isolation.
- [ ] **A14 — Distributed Token Bucket Rate Limiting:** Understand the Redis Lua token replenishment formula: `tokens = min(capacity, tokens + delta * refill_rate)`.
- [ ] **A15 — Semantic Caching Architecture:** Understand two-tier cache lookup: sub-1ms SHA-256 exact hash fast-path + vector cosine similarity (`distance <= 0.08`).
- [ ] **A16 — State-Machine Circuit Breaker Pattern:** Understand Michael Nygard's `CLOSED` -> `OPEN` -> `HALF_OPEN` state transitions and canary lease locks.
- [ ] **A17 — Exponential Backoff with Full Jitter:** Understand why full jitter eliminates thundering herd retry storms: `T ~ Uniform(0, min(max_wait, base * 2^attempt))`.
- [ ] **A18 — Cosine Similarity vs. Other Metrics & RRF Fusion:** Understand magnitude-invariance and Cormack et al.'s formula: `RRF_Score = sum(1 / (k + rank))`.
- [ ] **A19 — HNSW (Hierarchical Navigable Small World):** Understand multi-layer proximity graphs, `M=16`, `ef_construction=64/200`, and `ef_runtime=50`.
- [ ] **A20 — HMAC-SHA-256 Key Hashing & Constant-Time Comparison:** Understand why plaintext keys are never stored and how `hmac.compare_digest` prevents timing attacks.
- [ ] **A21 — Tenant Isolation & Anti-Spoofing:** Understand how `resolve_authorized_tenant()` cross-references caller-supplied tenant IDs against authenticated credentials.
- [ ] **A22 — LLM Streaming & Pre-First-Chunk Failover:** Understand why model failover is safe before the first emitted token, but impossible mid-stream.

### Phase 1 Completion Criteria
> **Done when:** You can close the document and verbally explain the difference between ASGI and WSGI, how the two-tier semantic cache functions, and why the circuit breaker uses a single canary probe during `HALF_OPEN`.

---

## Phase 2: Project Walkthrough (Codebase Grounding)

**Objective:** Map every conceptual model to real lines of code in the repository. Open the SentinelRAG source code on your laptop IDE and follow [`01_SentinelRAG_Project_Walkthrough.md`](./01_SentinelRAG_Project_Walkthrough.md).

### Phase 2 Action Checklist
- [ ] Read **Section 1 & 2** of [`01_SentinelRAG_Project_Walkthrough.md`](./01_SentinelRAG_Project_Walkthrough.md) to understand project structure and tech stack.
- [ ] Trace **Application Entry Point** in `app/main.py`:
  - [ ] Inspect `lifespan()` context manager (database & Redis pool initialization and clean teardown).
  - [ ] Verify middleware registration order (RateLimiter -> CORS -> Timing/Correlation).
  - [ ] Inspect `/healthz` dual dependency ping and `/metrics` Prometheus exposition.
- [ ] Trace **Configuration** in `app/core/config.py`:
  - [ ] Inspect `Settings` class defaults (`SEMANTIC_CACHE_THRESHOLD=0.08`, `CIRCUIT_BREAKER_BACKEND="local"`, `API_AUTH_ENABLED=False`).
  - [ ] Verify protocol validators for `DATABASE_URL` (`postgresql+asyncpg://`) and `REDIS_URL` (`redis://`).
- [ ] Trace **Database & ORM Layer**:
  - [ ] Inspect `DatabaseManager` and `get_db_session()` in `app/core/database.py` and `app/db/session.py`.
  - [ ] Inspect `DocumentChunk` model in `app/models/document.py` (Vector(1536), GIN index, computed `search_vector`).
- [ ] Trace **Security & Identity** in `app/core/security.py`:
  - [ ] Inspect `hash_api_key()` (HMAC-SHA-256 with pepper).
  - [ ] Trace `verify_api_key()` constant-time lookup and `resolve_authorized_tenant()` anti-spoofing logic.
- [ ] Trace **Rate Limiting** in `app/core/rate_limiter.py`:
  - [ ] Read `TOKEN_BUCKET_LUA_SCRIPT` line by line.
  - [ ] Trace `TokenBucketRateLimitMiddleware.dispatch()` and header injection.
- [ ] Trace **Resilience & Circuit Breaking** in `app/core/circuit_breaker.py`:
  - [ ] Trace process-local `CircuitBreaker` (`asyncio.Lock`).
  - [ ] Trace distributed `RedisCircuitBreaker` (`CB_CAN_EXECUTE_LUA` with canary probe lease lock).
  - [ ] Trace `execute_with_resilience()` in `app/services/resilience.py` (Tenacity retryer).
- [ ] Trace **Semantic Cache & Hybrid Retrieval**:
  - [ ] Inspect `PythonScanBackend` and `RediSearchBackend` in `app/services/cache.py`.
  - [ ] Trace exact-hash lookup `_exact_lookup()` vs vector lookup `lookup_vector()`.
  - [ ] Trace `SearchRepository.search_hybrid_candidates()` in `app/repositories/search.py` (sequential dense + sparse execution).
  - [ ] Trace `fuse_hybrid_candidates()` in `app/services/rrf.py` (`k=60`).
- [ ] Trace **The Unified Chat Endpoint** in `app/api/v1/chat.py` (The Heart of the System):
  - [ ] Trace **Stage 0:** Auth verification, tenant resolution, and prompt hygiene validation (`PromptValidator`).
  - [ ] Trace **Stage 1:** Two-tier cache lookup with lazy `vector_provider` embedding generation.
  - [ ] Trace **Stage 2:** Hybrid context retrieval and prompt augmentation (`_augment_messages_with_retrieval`).
  - [ ] Trace **Stage 3A:** SSE streaming path (`stream_complete`, `request.is_disconnected()`, `stream_bg_tasks`).
  - [ ] Trace **Stage 3B:** Buffered completion path with automatic provider failover (Groq -> OpenAI -> Mistral).
  - [ ] Trace **Stage 4:** Deferred cache persistence via `BackgroundTasks` and token/cost telemetry updates.
- [ ] Review **Section 24 (End-to-End Scenarios)** in [`01_SentinelRAG_Project_Walkthrough.md`](./01_SentinelRAG_Project_Walkthrough.md):
  - [ ] Scenario 1: Semantic Cache Hit (Exact vs. Vector Cosine).
  - [ ] Scenario 2: Cache Miss -> Hybrid Retrieval -> Primary LLM -> Background Cache Write.
  - [ ] Scenario 3: SSE Streaming Request with Client Disconnect Handling.
  - [ ] Scenario 4: Authentication & Tenant Spoofing Failure.
  - [ ] Scenario 5: Prompt Injection / Special Control Token Rejection.
  - [ ] Scenario 6: Primary Provider Outage & Circuit Breaker Failover.
  - [ ] Scenario 7: Rate Limit Quota Exhaustion (RFC 7807 429 Response).

### Phase 2 Completion Criteria
> **Done when:** You can trace an incoming HTTP request through `app/main.py` -> `app/api/v1/chat.py` -> Redis/pgvector/LiteLLM -> client response without looking at the notes.

---

## Phase 3: Core Concepts — Second Pass (Deep Systems Rigor)

**Objective:** Cement architectural depth, trade-offs, and supporting components. Revisit [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md) with an emphasis on **Category B (Supporting Concepts)** and selected **Category C (Systems Mechanics)**.

### Phase 3 Action Checklist
- [ ] Study **Category B Supporting Concepts** in [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md):
  - [ ] **B1 — `BaseModel` Internals (`pydantic-core` in Rust):** Schema compilation, lax vs strict mode, `@field_validator`.
  - [ ] **B2 — `BaseSettings` (`pydantic-settings`):** Dynamic environment variable loading, case sensitivity, bounds validation.
  - [ ] **B3 — `asyncio.wait_for()` & Timeout Enforcement:** Per-request timeouts on embedding generation and LLM calls.
  - [ ] **B4 — `AsyncIterator` and Async Generators:** Streaming generator cleanup and `finally` execution under cancellation.
  - [ ] **B5 — PostgreSQL `JSONB` and GIN Indexing:** Fast arbitrary document metadata storage and containment queries.
  - [ ] **B6 — Redis In-Memory Data Model:** String, Hash, and Set layout across SentinelRAG namespaces.
  - [ ] **B7 — Multi-Tenancy Architecture Patterns:** Shared-table (`tenant_id`) vs schema-per-tenant vs DB-per-tenant.
  - [ ] **B8 — Word Embeddings, Sentence Embeddings, and Dense Vectors:** Representation in 1536-dimensional space.
  - [ ] **B9 — Tokenization (`tiktoken` / BPE) & ChatML Overhead:** `+3` token framing overhead per message and offline fallback tokenizer.
  - [ ] **B10 — Prompt Injection & Input Hygiene:** Defense-in-depth regex rules, C0 control character stripping, and special control tokens.
  - [ ] **B11 — Fail-Open vs. Fail-Closed Security Posture:** Contrasting rate limiter (fail-open) with authentication (fail-closed).
  - [ ] **B12 — Structured Logging with `structlog`:** JSON rendering in production and contextual key-value logging.
  - [ ] **B13 — Prometheus Metrics & Observability:** Counters (`CACHE_OPERATIONS_TOTAL`), histograms (`HTTP_REQUEST_DURATION_SECONDS`).
  - [ ] **B14 — LiteLLM Multi-Provider Abstraction:** Universal interface for 100+ LLMs and model parameter normalization.
- [ ] Review **Staff/Principal Deep Dives** in Category C:
  - [ ] **C13–C16 (Systems Deep Dives):** RFC 7807 Problem Details, PostgreSQL MVCC & VACUUM, Redis memory layout & `fork()` Copy-on-Write, and TCP Backpressure.
  - [ ] **C17–C21 (Kernel & Storage Engine):** Linux `epoll` socket multiplexing, HTTP/1.1 vs HTTP/2 keepalive, and Redis Cluster hash slots.
  - [ ] **C22–C31 (Architectural Trade-Offs Matrix):** In-Postgres search vs Dedicated Vector DB (Pinecone/Milvus), RRF vs Cross-Encoder reranking, and Semantic Cache vs Provider Prompt Caching.

### Phase 3 Completion Criteria
> **Done when:** You can defend why SentinelRAG chose PostgreSQL pgvector over a standalone vector database like Pinecone, why RRF was chosen over linear score blending, and how Redis copy-on-write spikes impact memory sizing.

---

## Phase 4: Interview Questions and Answers (Active Verbal Practice)

**Objective:** Convert technical understanding into fluent, articulate interview delivery. Open [`03_SentinelRAG_Interview_QA.md`](./03_SentinelRAG_Interview_QA.md) and practice the **Category A Essential Questions (75 Questions)** aloud.

### Category A Essential Questions Practice Checklist

#### 1. Project Introduction & Architecture (8 Questions)
- [ ] **Q1:** What is SentinelRAG and what problem does it solve?
- [ ] **Q2:** Walk me through the architecture of SentinelRAG at a high level.
- [ ] **Q3:** What is the tech stack and why was each component chosen?
- [ ] **Q5:** What is multi-tenancy and how does SentinelRAG implement it?
- [ ] **Q6:** Explain the semantic caching strategy in SentinelRAG.
- [ ] **Q7:** What is hybrid retrieval and why does SentinelRAG use it?
- [ ] **Q8:** How does SentinelRAG handle LLM provider failures?
- [ ] **Q9:** Walk me through the entire lifecycle of a chat completion request in SentinelRAG.

#### 2. Source-Code Implementation (15 Questions)
- [ ] **Q11:** What does the `lifespan()` context manager in `main.py` do?
- [ ] **Q13:** In `chat.py`, what does `resolve_authorized_tenant()` do and why is it important?
- [ ] **Q14:** Explain the `_get_or_generate_vector()` inner closure in `chat_completions()`. Why is it a closure?
- [ ] **Q15:** In `cache.py`, what does the `PythonScanBackend.lookup_vector()` method do step by step?
- [ ] **Q16:** What does `compute_query_hash()` do, and why does it normalize the query before hashing?
- [ ] **Q17:** In `circuit_breaker.py`, what is the purpose of the `CB_CAN_EXECUTE_LUA` Lua script?
- [ ] **Q18:** Explain the `_persist_cache_entry()` function in `chat.py`. What is the significance of `CACHE_BACKGROUND_WRITES`?
- [ ] **Q19:** In `resilience.py`, what is `is_transient_error()` and how does it classify exceptions?
- [ ] **Q20:** What does `_extract_embedding()` in `embeddings.py` do? Why handle both dict and object shapes?
- [ ] **Q21:** In `circuit_breaker.py`, the `_eval_lua()` method has a `NOSCRIPT` fallback. Explain when this triggers.
- [ ] **Q22:** In `router.py`'s `stream_complete()`, why does the code set `emitted_any_token` flag and refuse mid-stream failover?
- [ ] **Q23:** Explain the defense-in-depth tenant assertion in `RediSearchBackend.lookup_vector()`.
- [ ] **Q24:** In `prompt_guard.py`, `_detect_special_tokens()` tries `tiktoken` with `allowed_special=set()` and catches `ValueError`. Explain this.
- [ ] **Q25:** In `cache.py`'s `get_or_lookup()`, why is `vector_provider` passed as a callable rather than resolving the vector upfront?
- [ ] **Q27:** Explain the `serialize_embedding()` / `deserialize_embedding()` pattern in `cache.py`. Why base64?

#### 3. Core Technical Concepts (15 Questions)
- [ ] **Q31:** What is the difference between ASGI and WSGI?
- [ ] **Q32:** How does Python's asyncio event loop handle multiple concurrent requests in a single Uvicorn worker?
- [ ] **Q33:** Why is `asyncio.Lock()` sufficient for thread safety in a FastAPI application?
- [ ] **Q35:** How do FastAPI's `BackgroundTasks` differ from `asyncio.create_task()` in terms of lifecycle and error handling?
- [ ] **Q36:** What is the significance of `structlog.contextvars` vs `threading.local` in an async FastAPI application?
- [ ] **Q40:** What is `pgvector` and what does the `<=>` operator do?
- [ ] **Q41:** Explain the HNSW algorithm and the `M` and `ef_construction` parameters in SentinelRAG's index.
- [ ] **Q42:** What is a `tsvector` generated column, and why is it persisted in SentinelRAG?
- [ ] **Q43:** Why does `search_hybrid_candidates()` run dense and sparse searches sequentially rather than concurrently?
- [ ] **Q48:** What is the token bucket algorithm and how does the Lua script in `rate_limiter.py` implement it?
- [ ] **Q49:** Why does SentinelRAG use Lua scripts for the rate limiter and circuit breaker instead of regular Redis commands?
- [ ] **Q50:** Explain the Redis `SMEMBERS` + pipeline `HGETALL` pattern in `PythonScanBackend`. What are its scalability limits?
- [ ] **Q51:** How does the RediSearch `FT.SEARCH` KNN query ensure tenant isolation?
- [ ] **Q55:** What is a timing attack and how does `hmac.compare_digest()` prevent it?
- [ ] **Q57:** What is tenant spoofing and how does `resolve_authorized_tenant()` prevent it?

#### 4. Why & What-If Questions (12 Questions)
- [ ] **Q61:** Why use Redis for semantic caching instead of an in-memory Python dict?
- [ ] **Q62:** Why use cosine similarity for semantic caching instead of exact string matching?
- [ ] **Q63:** Why is the cosine distance threshold set to 0.08? What happens if you set it to 0.5?
- [ ] **Q64:** Why does SentinelRAG use LiteLLM instead of calling OpenAI/Groq APIs directly?
- [ ] **Q65:** Why is RRF `k=60` the default? What happens with `k=1` vs `k=100`?
- [ ] **Q66:** Why use Tenacity for retry logic rather than a manual `try/except` loop?
- [ ] **Q67:** What would happen if `CACHE_BACKGROUND_WRITES=True` and the application crashes immediately after returning the response?
- [ ] **Q68:** Why does SentinelRAG's SSE streaming not support mid-stream provider failover?
- [ ] **Q69:** Why is the circuit breaker configured with `failure_threshold=5` and `recovery_timeout=60s`?
- [ ] **Q70:** What happens when the embedding service fails (`EmbeddingGenerationException`)? Trace graceful degradation.
- [ ] **Q72:** Why does SentinelRAG's `PythonScanBackend` become a bottleneck at scale? What is the complexity?
- [ ] **Q74:** Why does the RediSearch KNN query use a TAG pre-filter instead of a post-filter for tenant isolation?

#### 5. Scenario-Based Questions (10 Questions)
- [ ] **Q81:** A user reports that identical queries from two different tenants receive different cache hit rates. How do you investigate?
- [ ] **Q82:** The `/healthz` endpoint returns 503 (Redis is "down") but Redis is actually running. What would you check?
- [ ] **Q83:** A production deployment shows increasing response latency over time, stabilizing at ~400ms. What could cause this?
- [ ] **Q85:** After deploying a new embedding model (`EMBEDDING_MODEL`), cached responses are no longer being returned. Why?
- [ ] **Q86:** A client is sending `stream=True` requests but never sees the final `[DONE]` marker. What are the failure modes?
- [ ] **Q87:** The circuit breaker trips to `OPEN` but Groq's status page shows no outage. What could cause this?
- [ ] **Q90:** A malicious user submits a request with `query_vector=[0.0]*1536` (all zeros). What happens in the system?
- [ ] **Q91:** RediSearch `FT.SEARCH` returns results but the tenant assertion check in `lookup_vector()` triggers. What does this mean?
- [ ] **Q92:** During a Groq outage, requests failover to OpenAI. After 60s, one canary hits Groq, but subsequent requests still hit OpenAI for 60s. Explain this.
- [ ] **Q95:** The SSE streaming path shows that `stream_bg_tasks` background cache writes are being lost. When would this occur?

#### 6. Trade-Offs & Design Decisions (8 Questions)
- [ ] **Q96:** Python scan vs RediSearch HNSW: compare the two semantic cache backends in SentinelRAG.
- [ ] **Q97:** Local circuit breaker vs Redis distributed circuit breaker: compare trade-offs.
- [ ] **Q98:** Inline cache write vs background cache write: compare trade-offs.
- [ ] **Q99:** Dense-only retrieval vs hybrid (dense + sparse) retrieval: compare.
- [ ] **Q100:** SSE streaming vs buffered response: compare use cases and trade-offs.
- [ ] **Q101:** HNSW vs IVFFlat index in `pgvector`: when would you choose each?
- [ ] **Q102:** Semantic cache threshold 0.08 (strict) vs 0.20 (lenient): operational implications.
- [ ] **Q105:** Single-embedding architecture (one model for cache + retrieval) vs dual-embedding: trade-offs.

#### 7. Testing & Production Readiness (7 Questions)
- [ ] **Q106:** How is the FastAPI application tested without starting a real HTTP server?
- [ ] **Q108:** How does SentinelRAG's test suite isolate tests from actual Redis and PostgreSQL?
- [ ] **Q109:** Explain how `app.dependency_overrides` works in FastAPI testing. Give a SentinelRAG example.
- [ ] **Q114:** Explain the difference between testing a streaming endpoint and a regular JSON endpoint using httpx ASGI transport.
- [ ] **Q116:** How do you configure SentinelRAG for production vs development?
- [ ] **Q117:** What does the `/healthz` endpoint check and what does it return?
- [ ] **Q121:** How would you horizontally scale SentinelRAG to handle 10x the current load?

### Phase 4 Completion Criteria
> **Done when:** You can answer all 75 Category A questions out loud, following the Direct Answer -> Technical Explanation -> SentinelRAG Connection -> Follow-up structure without hesitation.

---

## Phase 5: Practical Code Familiarity & Live Debugging

**Objective:** Gain complete muscle memory and confidence in making code modifications, debugging failure scenarios, and running validation suites.

### Phase 5 Action Checklist
- [ ] **Run the Benchmark Suite:**
  ```powershell
  python scripts/benchmark.py --mode simulated --iterations 100
  ```
  - [ ] Understand why `semantic_scan_100_entries` takes ~368ms in pure Python ($O(N \cdot D)$) and motivates RediSearch.
  - [ ] Observe that RRF hybrid fusion takes sub-millisecond execution (~0.4ms) at 2,400+ RPS.
  - [ ] Observe that prompt guard validation executes in ~0.3ms with cached `tiktoken` tokenizers.
- [ ] **Run the Sync Test Suite:**
  ```powershell
  pytest -m "not asyncio and not live and not benchmark" -o asyncio_mode=strict -v
  ```
  - [ ] Confirm all 18 sync unit tests pass across embeddings validation, prompt guard heuristics, and security hashing.
- [ ] **Inspect Live Architecture Components:**
  - [ ] Verify `docker-compose.yml` services: `db` (`ankane/pgvector:v0.5.1`), `redis` (`redis:7.2-alpine` / `redis/redis-stack-server`), and `app`.
  - [ ] Inspect Nginx reverse proxy configuration in `nginx/conf.d/sentinel.conf` (`proxy_buffering off;`, `X-Accel-Buffering: no;`).
- [ ] **Hands-On Modification Exercises:**
  - [ ] *Exercise 1:* Trace what changes if `SEMANTIC_CACHE_THRESHOLD` is lowered from `0.08` to `0.04`.
  - [ ] *Exercise 2:* Trace what happens if `CIRCUIT_BREAKER_BACKEND` is flipped from `"local"` to `"redis"`.
  - [ ] *Exercise 3:* Trace what happens if an unauthenticated user sends a request when `ENVIRONMENT=production`.

### Phase 5 Completion Criteria
> **Done when:** You can navigate to any function in `app/`, explain its parameters and side effects, and accurately predict the operational consequences of changing any configuration variable in `app/core/config.py`.

---

## Progress Tracking Summary

| Phase | Core Objective | Primary Resource | Completion Status |
| :--- | :--- | :--- | :---: |
| **Phase 1** | Core Concepts — First Pass (22 Concepts) | [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md) | [ ] |
| **Phase 2** | Codebase Grounding (Full Walkthrough) | [`01_SentinelRAG_Project_Walkthrough.md`](./01_SentinelRAG_Project_Walkthrough.md) | [ ] |
| **Phase 3** | Core Concepts — Second Pass (Systems Rigor) | [`02_SentinelRAG_Core_Concepts.md`](./02_SentinelRAG_Core_Concepts.md) | [ ] |
| **Phase 4** | Interview Q&A Mastery (75 Questions) | [`03_SentinelRAG_Interview_QA.md`](./03_SentinelRAG_Interview_QA.md) | [ ] |
| **Phase 5** | Practical Code Familiarity & Benchmarks | [`scripts/benchmark.py`](../scripts/benchmark.py), `tests/` | [ ] |
