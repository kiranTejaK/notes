# Question Bank A: Project Overview & Request Lifecycle

> **How to Use This Bank:** Practice speaking **Layer 1 (Short Answer)** in 20–40 seconds out loud before expanding into **Layers 2–5** as the interviewer probes deeper.

---

## Q1: Explain SentinelRAG—what is it and what problem does it solve?

### Layer 1 – Short Answer (20–40s)
> "SentinelRAG is an asynchronous AI middleware gateway I built with Python, FastAPI, PostgreSQL 16 (`pgvector`), and Redis 7. It sits between client applications and external LLM providers like Groq and OpenAI to solve three production bottlenecks: redundant LLM inference cost and latency via semantic caching, poor keyword/ID recall in pure vector RAG via single-store hybrid retrieval, and cascading outages from vendor rate limits or downtime via circuit-broken multi-provider failover."

### Layer 2 – Detailed Explanation
Direct client-to-LLM integrations break down in four predictable ways:
1. **Repetitive Spend & Latency:** Autoregressive LLM decoding routinely takes hundreds of milliseconds to seconds and charges per token, yet enterprise users ask the same questions using slightly different phrasing. Exact-key caches miss paraphrases completely.
2. **Vector Blind Spots:** Dense embeddings map semantic intent well, but BPE tokenizers split exact SKUs, CVEs, or error codes (`ERR_CONN_RESET`), hurting exact lexical match accuracy.
3. **Database Sprawl:** Maintaining Postgres + Pinecone/Qdrant + Elasticsearch introduces dual-write drift whenever documents update or delete.
4. **Provider Fragility & Noisy Neighbors:** Without per-tenant rate limits, timeouts, jittered retries, and circuit breaking, one runaway client or vendor `429/503` brownout can exhaust worker pools across the whole service.

### Layer 3 – Project Connection
SentinelRAG unifies all four defenses behind a single OpenAI-compatible gateway endpoint (`POST /api/v1/chat/completions` in `app/api/v1/chat.py`), backed by `TokenBucketRateLimitMiddleware` (Redis Lua), `RedisSemanticCache` (SHA-256 + cosine distance `<= 0.08`), `SearchRepository` + `rrf.py` (PostgreSQL `pgvector` HNSW + `tsvector` GIN + RRF `k=60`), and `ResilientLLMRouter` (`CircuitBreaker` + `Tenacity` + `LiteLLM`).

### Layer 4 – Follow-up Questions
- *How did you decide what NOT to put in scope?*
  - Deliberately excluded heavy orchestration frameworks (LangChain/LlamaIndex) to keep stack traces clean and event-loop behavior deterministic, excluded multi-agent loops, and kept retrieval unified in PostgreSQL 16 rather than deploying external vector/search clusters.
- *What isn't implemented yet vs. what is?*
  - Currently, callers pass `query_vector` in the request payload (server-side auto-embedding on omission is a planned upgrade), responses are buffered JSON (SSE `stream=True` is planned—Nginx is already tuned for it), and the `CircuitBreaker` is worker-local rather than Redis-synchronized.

---

## Q2: Why did you build this project, and what was your individual contribution?

### Layer 1 – Short Answer
> "At KloudWorx I spent nearly three years building production FastAPI backends with MySQL, Redis, and AWS. As I transitioned toward AI platform engineering, I saw that calling an LLM API is trivial, whereas operating it reliably, affordably, and with high retrieval precision is a backend systems problem. I designed SentinelRAG end-to-end to apply distributed backend patterns—Lua atomicity, single-store hybrid indexing, state-machine circuit breaking, and structured telemetry—specifically to LLM infrastructure."

### Layer 2 – Detailed Explanation & Honest Framing
- **Ownership:** I architected the 4-stage pipeline, defined the PostgreSQL `DocumentChunk` schema and index strategy, designed the fail-open Redis Lua rate limiter and two-tier binary vector cache, built the `CircuitBreaker` + `Tenacity` + `LiteLLM` routing cascade, and created the 44-test async suite (`FakeRedis`/`AsyncMock`) plus multi-stage Docker/Nginx/GitHub Actions deployment definitions.
- **Honesty Check (Important):** Keep SentinelRAG clearly positioned as a self-built portfolio/GitHub systems engineering project (as shown on your resume as *In Progress*). Never claim it ran live B2B traffic at an employer; frame all latency/savings benchmarks as design SLOs and mocked test verifications.

---

## Q3: Walk me through the complete lifecycle of a `POST /api/v1/chat/completions` request.

### Layer 1 – Short Answer
> "An incoming HTTPS request terminates TLS at Nginx and enters Uvicorn/FastAPI. First, correlation middleware assigns an `X-Request-Id` and binds `structlog` `contextvars`, then CORS runs, and then an atomic Redis Lua token-bucket middleware enforces quotas. Inside `chat_completions`, Stage 1 checks our two-tier Redis semantic cache (SHA-256 fast path, then cosine distance `<= 0.08`). On a miss, Stage 2 runs sequential dense HNSW and sparse `tsvector` queries in PostgreSQL 16 and fuses them via RRF (`k=60`). Stage 3 calls Groq via LiteLLM protected by Tenacity full-jitter retries and a 3-state circuit breaker that fails over to OpenAI/Mistral. Stage 4 writes the result to Redis with a 24h TTL and updates Prometheus metrics."

### Layer 2 – Detailed Explanation (All 3 Branches)
1. **Ingress & Middleware (`app/main.py`, `app/core/rate_limiter.py`):**
   - Because Starlette wraps middlewares outside-in, `correlation_id_and_timing_middleware` executes *first* (`X-Request-Id`, `clear_contextvars()`, `time.perf_counter()`), followed by `CORSMiddleware`, followed by `TokenBucketRateLimitMiddleware` (`EVALSHA` Lua check on `X-API-Key` or IP, returning RFC 7807 `429` if empty, or failing open if Redis is down).
2. **Branch A — Cache Hit (`app/services/cache.py`):**
   - `get_or_lookup` normalizes the last `user` message, checks SHA-256 (`HGETALL cache:{tenant}:{hash}`), and if absent runs a `SMEMBERS` + pipelined `HGETALL` scan decoding base64 IEEE 754 floats and testing `distance <= 0.08`.
   - On hit: counts saved tokens via `tiktoken`, increments Redis `dollars_saved` + Prometheus `COST_SAVED_USD_TOTAL` + `CACHE_OPERATIONS_TOTAL(result="hit")`, and returns HTTP 200 with `cost_usd=0.0`—skipping Postgres and LLMs completely.
3. **Branch B — Cache Miss + Primary LLM Success:**
   - If `use_retrieval=True`, `SearchRepository` sequentially executes dense `<=>` (HNSW) and sparse `websearch_to_tsquery` + `ts_rank_cd` (GIN) on one `AsyncSession`, `fuse_hybrid_candidates(k=60)` picks `top_k_chunks`, and injects a `system` message at index 0.
   - `ResilientLLMRouter.complete` checks `circuit_breaker.can_execute()` (`CLOSED`), calls Groq `llama-3.3-70b-versatile` via `execute_with_resilience` (`max_attempts=3`, `timeout=4.0s`), records circuit success + token cost, caches response (`86400s` TTL), and records Prometheus metrics.
4. **Branch C — Cache Miss + Circuit Breaker Failover:**
   - If Groq exhausts retries 5 consecutive times, `CircuitBreaker` trips `CLOSED` $\to$ `OPEN` for `60s`.
   - While `OPEN`, `can_execute()` returns `False` in `0ms`, routing straight to `openai/gpt-4o-mini` (`max_attempts=2`), with `mistral/mistral-small-latest` as tertiary fallback. After 60s, one `HALF_OPEN` canary probe tests recovery.

### Layer 4 – Follow-up Questions
- *Why doesn't the rate limiter run before correlation middleware?*
  - In `app/main.py`, `@app.middleware("http")` wraps outside previously added middlewares, so `correlation_id_and_timing_middleware` runs first—which is why even `429` rejections carry `X-Request-Id` and get counted in `HTTP_REQUEST_DURATION_SECONDS`.
- *What happens if PostgreSQL fails during Stage 2 hybrid search inside `chat_completions`?*
  - Stage 2 is wrapped in `try/except`; it logs `hybrid_retrieval_degraded_continuing_unaugmented` and lets Stage 3 answer without retrieved context rather than failing the chat completion outright.

---

## Q4: What are all the major components and API endpoints in SentinelRAG?

### Layer 1 – Short Answer
> "Beyond `/api/v1/chat/completions`, the gateway exposes batch chunk ingestion (`POST /api/v1/documents/batch`), standalone hybrid retrieval (`POST /api/v1/search/hybrid`), semantic cache management (`/api/v1/cache/evaluate`, `/set`, `/invalidate`), Kubernetes/Docker health probing (`GET /healthz`), Prometheus scraping (`GET /metrics`), and an operational Streamlit dashboard."

### Layer 3 – Component & Route Map
- `app/api/v1/documents.py`: Bulk `INSERT ... RETURNING id` up to `1000` chunks (`201 Created`).
- `app/api/v1/search.py`: Standalone HNSW + FTS + RRF query endpoint returning ranked chunks with dense/sparse/fused scores.
- `app/api/v1/cache.py`: Cache threshold testing (`evaluate`), warming (`set`), and tenant flush (`invalidate`).
- `app/main.py`: `/healthz` (`200` when Postgres + Redis ping succeed, `503` when degraded) and `/metrics` (OpenMetrics text).
- `dashboard/app.py`: Streamlit UI calling `/api/v1/chat/completions` to visualize latency, cache hits, and dollars saved.

---
**Next:** [B-architecture-and-design.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/B-architecture-and-design.md)

