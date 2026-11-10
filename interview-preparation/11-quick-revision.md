# 11. Quick Revision Cheat Sheet (Final 2 Hours)

Use this document for rapid mental warm-up in the final 1–2 hours before a technical interview. Every constant, file path, route, and behavior below is verified against [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md) and linked to the deeper guides for quick lookup.

---

## 1. Elevator Pitches (Rehearse Until Effortless)

### 30-Second Pitch
> "I built **SentinelRAG**—an asynchronous, multi-tenant AI gateway and hybrid RAG platform in Python and FastAPI. It sits between client applications and LLM providers to solve three production bottlenecks: cost, latency, and reliability. It protects upstream capacity with an atomic Redis Lua token bucket, eliminates redundant LLM calls via a two-tier SHA-256 and cosine-similarity semantic cache, retrieves grounded context using a single-store PostgreSQL hybrid engine combining `pgvector` HNSW and GIN full-text search via Reciprocal Rank Fusion, and routes completions through a circuit-broken Groq-to-OpenAI-to-Mistral fallback cascade with `tiktoken` token/cost telemetry."

### 2-Minute Engineering Pitch
1. **Why I built it:** Standard naive RAG pipelines overfit to dense embeddings (missing exact IDs/acronyms), pay full LLM latency and token cost on slightly rephrased repeat queries, and crash hard when an external LLM API throttles or times out.
2. **Edge & Governance:** Every request passes through Nginx (`proxy_buffering off` for streaming readiness) and FastAPI's ordered middleware stack—first stamping an `X-Request-ID` in `contextvars` for structured JSON logging (`structlog`) and Prometheus latency histograms, then enforcing per-tenant rate limits (`100` burst, `10/s` refill) via an atomic **Redis Lua token bucket** using Redis server `TIME` (failing open if Redis goes down).
3. **Semantic Cache:** Next, `RedisSemanticCache` checks for an exact **SHA-256** normalized query match, and if `query_vector` is present, scans tenant embedding hashes (`SMEMBERS` + pipelined `HGETALL`) packed as raw IEEE-754 floats in Base64, returning cached answers when cosine distance is $\le 0.08$ (similarity $\ge 0.92$).
4. **Hybrid Search + RRF:** On cache miss, `SearchRepository` queries **PostgreSQL 16 + `pgvector`**—sequentially running dense cosine search over an HNSW index (`m=16, ef_construction=64`) and sparse BM25-style ranking (`websearch_to_tsquery` + `ts_rank_cd`) over a persisted `tsvector` GIN index—and merges both candidate lists in Python using scale-invariant **Reciprocal Rank Fusion ($k=60$)**.
5. **Resilient Routing & Honesty Boundary:** The grounded prompt is sent to `ResilientLLMRouter` wrapping **LiteLLM**: primary `groq/llama-3.3-70b-versatile` (3 full-jitter retries, 4s timeout) protected by a 3-state async `CircuitBreaker` (5 failures $\to$ 60s open cooldown), automatically failing over to `openai/gpt-4o-mini` (2 retries) and `mistral/mistral-small-latest` (1 direct call). It is an active personal engineering project (44 unit/API tests with mocked LLM/Redis backends) where several Phase 2 upgrades—RediSearch `FT.SEARCH`, internal embedding generation, true SSE streaming, and distributed Redis circuit breakers—are explicitly scoped.

---

## 2. 5-Box Whiteboard Architecture (Draw in 90 Seconds)

```text
Client (X-Tenant-ID, query, query_vector)
  │
  ▼
[1. Edge & ASGI Gateway]  Nginx (TLS 1.3, proxy_buffering off)
  │                       └─► FastAPI MW Order: CorrelationID -> CORS -> RateLimiter
  ├──► [Redis 7.2]        Atomic Lua Token Bucket (100 burst, 10/s) + Fail-Open
  │                       2-Tier Cache: SHA-256 L1 -> Cosine <= 0.08 L2 (TTL 24h)
  ▼ (Cache Miss)
[2. Hybrid Retrieval]     PostgreSQL 16 + asyncpg (pool 20+10)
  │                       ├─ Dense: pgvector HNSW (m=16, ef_c=64, <=>)
  │                       └─ Sparse: GIN persisted tsvector + ts_rank_cd
  │                       └─► RRF Fusion (k=60, tie-break dense -> sparse)
  ▼
[3. Resilient LLM Router] LiteLLM + Tenacity Full Jitter + Async CircuitBreaker
  │                       Groq (3x, CB 5 fail/60s) -> OpenAI (2x) -> Mistral (1x)
  ▼
[4. FinOps & Observability] tiktoken (cl100k_base) + Structlog JSON + /metrics
```

Full architectural contract: [02-project-architecture.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/02-project-architecture.md) · Complete trace: [03-end-to-end-workflow.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/03-end-to-end-workflow.md).

---

## 3. Verified Codebase Facts & Exact Numbers Table

| Subsystem | Verified Exact Value / Symbol | Source Module |
| :--- | :--- | :--- |
| **Database Pool** | `pool_size=20`, `max_overflow=10`, `pool_timeout=30`, `pool_recycle=300`, `pool_pre_ping=True`, `expire_on_commit=False`, `autoflush=False` | [06/01-app-lifecycle](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/01-app-lifecycle-and-middleware.md) |
| **Redis Connection** | `max_connections=50`, `socket_timeout=0.5s`, `decode_responses=True`, `maxmemory 512mb`, `volatile-lru` | [05/03-redis-lua](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/03-redis-lua-and-semantic-caching.md) |
| **Middleware Runtime Order** | 1. `correlation_id_and_timing_middleware` $\to$ 2. `CORSMiddleware` $\to$ 3. `TokenBucketRateLimitMiddleware` | [06/01-app-lifecycle](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/01-app-lifecycle-and-middleware.md) |
| **Rate Limiter** | Capacity `100`, refill `10/sec` (`config.py` defaults); key `ratelimit:{tenant_id}`; TTL `ceil(cap/refill)*2`; exempts `/healthz`, `/`, `/docs`, `/redoc`, `/openapi.json` | [06/02-rate-limiter](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md) |
| **Semantic Cache** | Hash: **SHA-256** (`query.strip().lower()`); threshold distance $\le 0.08$ (similarity $\ge 0.92$); TTL `86400s`; `1536` dims; `struct.pack(f"{len(vector)}f", *vector)` + Base64 | [06/02-rate-limiter](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md) |
| **Vector Index** | `pgvector` cosine distance `<=>`; **HNSW** `m=16, ef_construction=64, vector_cosine_ops` on `DocumentChunk.embedding` (`Vector(1536)`) | [06/03-hybrid-search](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md) |
| **Full-Text Index** | Generated column `Computed("to_tsvector('english', chunk_text)", persisted=True)` + **GIN**; `websearch_to_tsquery('english', ...)` + `ts_rank_cd` | [06/03-hybrid-search](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md) |
| **RRF Fusion** | `score = sum(1 / (k + rank))`, `k=60`; tie-break `(-rrf_score, -dense_score, -sparse_score)`; `/search/hybrid` fetches `50+50` $\to$ `10`; `/chat/completions` fetches `top_k_chunks * 2` (default `6+6`) $\to$ `3` | [06/03-hybrid-search](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md) |
| **LLM Router Cascade** | Primary `groq/llama-3.3-70b-versatile` (3 attempts, max wait `4.0s`) $\to$ `openai/gpt-4o-mini` (2 attempts, max wait `3.0s`) $\to$ `mistral/mistral-small-latest` (1 attempt) | [06/04-router-chat](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/04-resilient-router-chat-and-analytics.md) |
| **Circuit Breaker** | `failure_threshold=5`, `recovery_time_seconds=60`, `timeout_seconds=4.0`; `asyncio.Lock` per provider | [06/04-router-chat](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/04-resilient-router-chat-and-analytics.md) |
| **API Endpoints** | `POST /api/v1/chat/completions`, `POST /api/v1/documents/batch` (max `1000` chunks, `201`), `POST /api/v1/search/hybrid`, `POST /api/v1/cache/{evaluate,set,invalidate}`, `GET /healthz` (`200`/`503`), `GET /metrics` | [03-end-to-end-workflow.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/03-end-to-end-workflow.md) |
| **Prometheus Metrics** | `http_request_duration_seconds` (13 buckets: `5ms`–`10s`), `cache_operations_total`, `tokens_consumed_total`, `cost_saved_usd_total` | [05/05-observability](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/05-observability-testing-and-infra.md) |
| **Container & CI/CD** | Multi-stage `python:3.11-slim` + `uv`, non-root `appuser` (`UID/GID 10001`), `--workers 1`, 3-job GitHub Actions (`test`, `build-and-push`, `deploy`) | [05/05-observability](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/05-observability-testing-and-infra.md) |
| **Test Suite Scope** | `44` tests across `12` test modules using `pytest-asyncio`, `ASGITransport`, `FakeRedis`, `AsyncMock` (no live Postgres/Redis/LLM in unit CI; `docker-compose` integration check is optional) | [05/05-observability](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/05-observability-testing-and-infra.md) |

---

## 4. Critical Code Gotchas & "Interviewer Trap" Catch List

If an interviewer asks you to trace the actual code—or spots a subtlety—you gain huge senior credibility by proactively knowing these 8 real edge cases in the repository:

1. **Starlette Middleware Reversal:** `@app.middleware("http")` wraps *outside* earlier `app.add_middleware(...)` calls. That is why `correlation_id_and_timing_middleware` runs **before** `CORSMiddleware` and `TokenBucketRateLimitMiddleware` (allowing the rate limiter to read `request.state.request_id`).
2. **Sequential DB Queries in `SearchRepository`:** `search_dense` and `search_sparse` are `await`ed **sequentially** on the single injected `AsyncSession`—never `asyncio.gather`ed—because SQLAlchemy's `AsyncSession` wraps one `asyncpg` connection whose protocol rejects concurrent queries (`InterfaceError`).
3. **SHA-256 + Python Scan (Not RediSearch):** `RedisSemanticCache` does **not** use RediSearch `FT.SEARCH`. It does L1 `HGETALL` by SHA-256 hash, then `SMEMBERS cache:{tenant_id}:keys` + pipelined `HGETALL` + lazy `SREM` for expired ghosts + Python cosine math. Always describe this accurately and cite RediSearch HNSW as the >5,000-key/tenant upgrade.
4. **Caller-Supplied `query_vector` & Zero-Vector Fallback:** `chat.py` takes an optional `query_vector` from the client. If omitted, L2 semantic cache similarity is skipped and `hybrid_search` falls back to `[0.0] * 1536`! Also, `cache_service.set()` runs inline before returning rather than in `BackgroundTasks`, and `stream=True` validates cleanly in Pydantic while `chat.py` still returns a buffered JSON `ChatResponse`.
5. **Half-Open Thundering Probe:** `CircuitBreaker` guards transitions with `asyncio.Lock`, which releases across the provider's `await` call. If state switches to `HALF_OPEN` after 60s under heavy load, multiple concurrent coroutines can read `HALF_OPEN` before the first trial completes (`_probe_in_flight` flag is the clean fix).
6. **`tiktoken` Special-Token Caveat:** `token_analytics.py` calls `self._encoder.encode(text)` without `allowed_special=set()`. Adversarial prompts containing literal `<|endoftext|>` will raise a `ValueError` unless escaped or disabled via `disallowed_special=()`.
7. **`--workers 1` Constraint:** Single-worker Uvicorn keeps `CircuitBreaker` state consistent and Prometheus `/metrics` accurate without multiprocess collectors. Scaling horizontally requires Redis-backed shared breaker state (`INCR`/`EXPIRE` + Lua) and an external rate-limit / metrics aggregator.
8. **Streamlit Dashboard Mock Fallback:** `dashboard/app.py` parses live `/metrics` text if reachable, but silently falls back to `use_mock = True` and uses heuristic multipliers to split cache vs. DB vs. LLM latency bars—never cite dashboard visuals as real production measurements.

---

## 5. 1-Line Design Decision Defenses

| Choice Made | Why Chosen Over Alternative | Primary Trade-Off / Upgrade Trigger |
| :--- | :--- | :--- |
| **PostgreSQL + `pgvector`** vs. Pinecone + ES | Transactional consistency between tenant ACLs/metadata and vectors; one database to back up and migrate | Upgrade above ~10M+ vectors/node or >5k hybrid QPS to dedicated Qdrant/OpenSearch |
| **HNSW** vs. IVFFlat | Better recall/latency curve without needing `ANALYZE`/re-clustering as documents stream in incrementally | ~2–3x RAM overhead; slow index builds—move heavy ingestion to offline/async workers |
| **RRF ($k=60$)** vs. Weighted Linear Sum | Dense cosine scores (`[0, 1]`) and Postgres `ts_rank_cd` (unbounded) aren't comparable; RRF uses pure rank order | Ignores absolute score magnitude; add a cross-encoder reranker if top-3 precision lags |
| **Redis Lua Token Bucket** vs. Sliding Window Log | Fixed $O(1)$ memory (1 hash/tenant) and atomic read-refill-decrement without multi-node clock skew via Redis `TIME` | Fail-open risks backend overload during simultaneous Redis outage + DDoS |
| **`asyncpg` + FastAPI** vs. Sync Django/Flask | Event-loop concurrency multiplexes 100s of slow LLM/DB awaits without thread-stack RAM bloat | Requires `pool_pre_ping`, sequential queries per session, and strict non-blocking handlers |
| **PgBouncer + `statement_cache_size=0`** | Avoids PostgreSQL process memory exhaustion (~10MB/backend) under high worker fan-out | Disables `asyncpg` prepared statement cache in transaction pooling mode |

Details & threshold triggers: [09-design-decisions-and-tradeoffs.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md).

---

## 6. Phase 2 Ultra-Quick Flashcards (KloudWorx & DOit)

Before stepping into behavioral/resume rounds, review these concrete anchors in [14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md):

- **KloudWorx FastAPI/Slim Strangler Fig:** Migrated I/O-heavy/CPU endpoints selectively behind an Nginx/API gateway without downtime, sharing JWT signing keys + MySQL schema.
- **SQS + Idempotency + DLQ:** Producer attaches `UUIDv4` idempotency key; consumer claims `SET key 1 NX EX 300` in Redis before executing; `WaitTimeSeconds=20` long polling; Redrive policy moves poison messages to DLQ after `maxReceiveCount=3`.
- **S3 Pre-Signed Uploads + Security:** Direct client-to-S3 (`ExpiresIn=900`, `AES256`) removes binary bytes from FastAPI RAM; worker validates **magic bytes** (`%PDF-`, `\x89PNG`) via `Range='bytes=0-7'` instead of trusting `Content-Type`.
- **MySQL Indexing & Concurrency:** Slow query log (`0.5s`) + `EXPLAIN ANALYZE`; composite B+Tree column ordering via **Equality $\to$ Sort $\to$ Range (ESR)**; InnoDB deadlock mitigation via deterministic lock ordering + small transactions + optimistic `version` checks.
- **Stripe Webhooks:** Verify `Stripe-Signature` HMAC-SHA256 + timestamp delta $< 300\text{s}$ (constant-time compare), insert `event.id` into `processed_webhooks` (PK constraint), return `200 OK` within 5s, process async via SQS.
- **Elasticsearch Nested Docs:** `nested` indexes each child object as a hidden Lucene document (preventing cross-object false positives from array flattening); wrap queries in `nested` query clause.
- **Redis Cache-Aside Invalidation:** Mutate DB first, then `DEL` cache key (never blind `SET` from stale concurrent readers); protect hot keys with `SET NX` singleflight / jittered TTL.
- **M.Sc. Chemistry $\to$ Backend Pivot:** Frame chemistry as hypothesis-driven root-cause analysis, quantitative precision, and methodical debugging (tested cleanly via CCBP TA mentorship $\to$ KloudWorx production ownership $\to$ SentinelRAG systems design).

---

## 7. 2-Minute Rapid Pass/Fail Self-Check

Answer out loud in **$\le 20$ seconds each** without looking up:

1. Which middleware executes **first** on an incoming HTTP request in SentinelRAG, and why? *(Answer: `correlation_id_and_timing_middleware` — Starlette wraps middlewares LIFO.)*
2. What exact two lookups happen inside `RedisSemanticCache.evaluate()`? *(Answer: Exact SHA-256 `HGETALL`, then if `query_vector` is provided, `SMEMBERS` + pipelined `HGETALL` + Python cosine scan $\le 0.08$.)*
3. Why must `SearchRepository` NOT `asyncio.gather(search_dense, search_sparse)` on its single `AsyncSession`? *(Answer: `asyncpg` binds one wire connection per `AsyncSession`; concurrent awaits raise `InterfaceError`.)*
4. How does RRF prevent BM25 scores from drowning out cosine scores? *(Answer: Converts both lists to $1/(k+\text{rank})$ with $k=60$, making fusion scale-invariant.)*
5. Walk through the exact fallback chain when Groq throws 5 consecutive `503`s. *(Answer: `groq` breaker opens for `60s`; next request skips Groq immediately in `Router`, tries OpenAI up to 2 full-jitter attempts, then Mistral once.)*
6. What happens to `/api/v1/chat/completions` if Redis completely dies? *(Answer: Token bucket logs/warns and fails open; cache misses cleanly; request still serves via Postgres + LLM.)*
7. Why did `Dockerfile` pick `--workers 1` and `appuser` `10001`? *(Answer: Exec-form PID 1 propagates `SIGTERM` cleanly, single worker shares process-local `CircuitBreaker`/Prometheus state, UID `10001` prevents container root escalation.)*
8. Why does Stripe webhook verification check both HMAC and timestamp tolerance? *(Answer: HMAC proves origin/integrity; checking $|t_{\text{now}} - t| < 300\text{s}$ stops replay attacks.)*
9. In MySQL composite index `(tenant_id, created_at, status)`, which queries cannot use `created_at` efficiently if filtering by `status`? *(Answer: Equality `tenant_id` + Equality `status` + Range `created_at` wants `(tenant_id, status, created_at)` — ESR rule!)*
10. What happens in SentinelRAG if a client passes `query_vector=None` to `/api/v1/chat/completions`? *(Answer: Semantic L2 vector search is skipped, and `hybrid_search` falls back to a zero vector `[0.0]*1536` — explicitly flag as a Phase 2 improvement to auto-embed server-side.)*

