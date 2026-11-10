# 01. Project Overview: SentinelRAG

**Project:** SentinelRAG — Multi-Tenant Enterprise AI Gateway & Hybrid Retrieval Engine  
**Tech Stack (Verified):** Python 3.11/3.12, FastAPI, Uvicorn (`uvloop`), Pydantic v2, SQLAlchemy 2.0 (Async), `asyncpg`, PostgreSQL 16 (`pgvector`, `tsvector`), Redis 7.2 (Lua scripting), LiteLLM, Tenacity, `tiktoken`, `structlog`, `prometheus-client`, Streamlit, Docker, Nginx, GitHub Actions, `pytest-asyncio`  
**Status Note (Resume Alignment):** Listed on your resume as an **In-Progress / self-initiated engineering project** (`GitHub Repository`). All code modules, unit/mocked integration tests, Docker/Nginx configs, and CI/CD workflows are documented in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md). Any live production traffic metrics cited in older notes (such as "54.2% billing drop" or "500 B2B customers") are treated as **target SLOs / unverified synthetic estimates** rather than live production telemetry.

---

## 1. What Problem SentinelRAG Solves

When backend applications call LLM APIs (OpenAI, Groq, Mistral) and naive vector databases directly, five concrete engineering bottlenecks appear:

1. **Redundant Inference Cost & Latency:**
   - Natural-language enterprise queries frequently repeat in paraphrased forms (*"How do I reset my password?"* vs. *"Password reset steps?"*). Direct LLM calls take hundreds of milliseconds to multiple seconds and charge per token on every repeat.
   - Standard exact-string HTTP/KV caches miss paraphrases completely.
2. **Pure Vector Search Blind Spots (Lexical Failure):**
   - Dense embeddings capture conceptual meaning (*"car repair"* $\leftrightarrow$ *"automobile maintenance"*), but BPE tokenization fragments exact identifiers—error codes (`ERR_CONN_RESET`), CVE IDs (`CVE-2024-38077`), SKUs, and part numbers—causing semantic near-misses instead of exact matches.
3. **Multi-Database Architectural Sprawl & Dual-Write Drift:**
   - Pairing PostgreSQL (for tenant/document metadata) with a dedicated vector store (Pinecone/Qdrant) and a separate lexical cluster (Elasticsearch) forces dual-writes across independent failure domains. If one write succeeds and another fails, retrieval returns orphaned or stale context unless you build outbox/CDC infrastructure.
4. **Upstream Provider Fragility:**
   - LLM APIs enforce strict RPM/TPM rate limits (`HTTP 429`) and suffer regional brownouts (`HTTP 503`, hanging sockets). Without timeouts, jittered retries, and circuit breaking, slow upstream sockets stall worker concurrency and cascade into gateway-wide outages.
5. **Noisy-Neighbor Quota Exhaustion & Black-Box Costs:**
   - In multi-tenant platforms, one runaway client script can exhaust shared LLM quotas or trigger upstream `429`s for every other tenant, while unstructured logs hide per-tenant token spend and stage-by-stage latency.

---

## 2. What SentinelRAG Is (Multi-Audience Explanations)

### 30-Second Elevator Pitch (Screening Calls)
> *"SentinelRAG is an asynchronous AI middleware gateway I engineered in Python, FastAPI, PostgreSQL 16, and Redis 7. It sits between client applications and LLM providers like Groq and OpenAI to solve cost, retrieval accuracy, and reliability. It enforces atomic Redis Lua rate limiting per tenant, checks a two-stage semantic cache (SHA-256 exact match plus cosine-distance vector matching) to serve repeat intents in milliseconds, runs single-store hybrid retrieval combining `pgvector` HNSW and PostgreSQL full-text search via Reciprocal Rank Fusion, and fails over automatically across providers using Tenacity retries and a coroutine-safe circuit breaker."*

### 2-Minute Deep Pitch (Technical Lead / Hiring Manager)
> *"Most AI integrations call an LLM SDK directly from application routes, which couples uptime to a single vendor, pays full token cost on repetitive questions, and struggles when vector search misses exact identifiers like SKUs or error codes.*
>
> *I designed SentinelRAG as an edge-to-LLM gateway following a four-stage pipeline on `POST /api/v1/chat/completions`:*
> *First, ingress middleware attaches a correlation ID via `structlog` `contextvars` and enforces per-key/per-IP token-bucket rate limits via an atomic Redis Lua script that uses Redis server `TIME` to eliminate worker clock skew.*
> *Second, a two-stage Redis cache checks a normalized SHA-256 key in $O(1)$, and on a miss scans tenant-isolated, base64-packed IEEE 754 vectors for cosine distance $\le 0.08$, skipping database retrieval and LLM inference completely on a hit while logging simulated dollar savings via `tiktoken`.*
> *Third, on a cache miss with retrieval enabled, SentinelRAG avoids dual-database sprawl by querying a single PostgreSQL 16 `document_chunks` table sequentially for `pgvector` HNSW dense candidates (`<=>`) and GIN-indexed `tsvector` lexical candidates (`ts_rank_cd`), merging both ranked lists with parameter-free Reciprocal Rank Fusion ($k=60$).*
> *Fourth, the augmented prompt is dispatched through a custom `ResilientLLMRouter` wrapping LiteLLM with Tenacity full-jitter retries and a three-state `CircuitBreaker`. After 5 consecutive primary failures on Groq, the breaker opens for 60 seconds and fails fast to OpenAI `gpt-4o-mini` (with Mistral as tertiary fallback). Everything is instrumented with Prometheus histograms/counters, tested via 44 mocked async `pytest` cases, and packaged behind Nginx in a non-root multi-stage Docker build."*

---

## 3. Major Components at a Glance

| Layer | Component | Verified Source Module | Core Responsibility |
| :--- | :--- | :--- | :--- |
| **Edge Proxy** | Nginx 1.27 | `nginx/nginx.conf`, `nginx/conf.d/sentinel.conf` | TLS 1.2/1.3 termination, HSTS, OCSP stapling, 50MB max body, `proxy_buffering off` for streaming readiness. |
| **ASGI Ingress** | FastAPI + Uvicorn (`uvloop`) | `app/main.py`, `app/core/config.py` | Lifespan pool warm-up/pings, Pydantic v2 `BaseSettings`, `/healthz`, `/metrics`, RFC 7807 error handler. |
| **Middleware** | Correlation + Token Bucket | `app/main.py`, `app/core/rate_limiter.py` | `X-Request-Id` context binding, latency histogram, atomic Redis Lua rate limit (`capacity=100`, `refill=10/s`). |
| **Caching** | `RedisSemanticCache` | `app/services/cache.py`, `app/api/v1/cache.py` | Stage 1 SHA-256 fast path; Stage 2 IEEE 754 binary vector cosine scan ($\le 0.08$); tenant key isolation & lazy eviction. |
| **Storage & Search** | `DocumentChunk` + `SearchRepository` | `app/models/document.py`, `app/repositories/search.py` | Single-store Postgres 16 table with HNSW (`m=16, ef_construction=64`), computed `TSVECTOR` GIN, JSONB GIN. |
| **Rank Fusion** | RRF Service | `app/services/rrf.py` | `reciprocal_rank_fusion` & `fuse_hybrid_candidates` ($k=60$) with deterministic dense/sparse tie-breaking. |
| **LLM Resilience** | Router + Breaker + Tenacity | `app/services/router.py`, `app/core/circuit_breaker.py`, `app/services/resilience.py` | `CLOSED`/`OPEN`/`HALF_OPEN` FSM (`asyncio.Lock`), Full Jitter retries, Groq $\to$ OpenAI $\to$ Mistral cascade. |
| **Economics & UI** | Analytics + Telemetry + Streamlit | `app/services/token_analytics.py`, `app/core/telemetry.py`, `dashboard/app.py` | `tiktoken` BPE counting, Redis `INCRBYFLOAT` spend/savings, 4 Prometheus metrics, Streamlit demo cockpit. |

---

## 4. Honest Boundary: Verified Implemented vs. Partial / Planned

Interviewers immediately respect engineers who know what their system **does**, what it **assumes**, and what the **next engineering steps** are:

### Verified in Codebase
- Asynchronous FastAPI gateway with DB/Redis pool pre-flight checks and `/healthz` (`200` vs `503`).
- Atomic Redis Lua Token Bucket middleware (`EVALSHA` + `NoScriptError` fallback + fail-open on Redis errors).
- Two-stage `RedisSemanticCache` (SHA-256 + pipelined cosine scan, IEEE 754 `struct.pack` + Base64, 24h TTL, lazy `SREM` cleanup, tenant invalidation).
- PostgreSQL 16 + `pgvector` single-store `document_chunks` model, Alembic migrations, batch `INSERT ... RETURNING`, sequential dense (`<=>`) + sparse (`websearch_to_tsquery` + `ts_rank_cd`) repository, and RRF ($k=60$).
- Coroutine-safe `CircuitBreaker` (`CLOSED`/`OPEN`/`HALF_OPEN`), Tenacity Full Jitter transient retry filter, LiteLLM router (Groq $\to$ OpenAI $\to$ Mistral), `tiktoken` analytics, `structlog` JSON logging, Prometheus `/metrics`.
- 44 async unit & mocked integration tests (`FakeRedis`, `AsyncMock`, `ASGITransport`), Streamlit dashboard, multi-stage `uv` Dockerfile (`UID 10001`), Nginx config, and 3-job GitHub Actions CI/CD YAML.

### Known Gaps, Assumptions & Planned Extensions (Be Ready to Discuss!)
1. **Caller-Supplied `query_vector`:** `ChatCompletionRequest` accepts an optional `query_vector: list[float] | None` rather than calling an embedding API inside the gateway. *Gap in current code:* if `query_vector` is omitted on a cache miss, `chat.py` falls back to `[0.0] * 1536` for dense retrieval and raises a caught exception in Stage 4 `cache_service.set()`. **Fix:** generate embeddings server-side via `litellm.aembedding()` when `query_vector` is `None`.
2. **Synchronous vs. Background Cache Write:** Stage 4 awaits `cache_service.set()` inline before returning the response. Moving it to FastAPI `BackgroundTasks` would shave a Redis RTT off cache-miss response latency.
3. **Python Scan vs. RediSearch `FT.SEARCH`:** Current semantic lookup pulls tenant keys via `SMEMBERS` + pipelined `HGETALL` and scores them in Python ($O(N \cdot D)$). Great for vanilla Redis 7 up to a few thousand keys per tenant; should migrate to Redis Stack `FT.SEARCH` HNSW for larger caches.
4. **Buffered JSON Only (No Active SSE Route Yet):** Nginx is tuned with `proxy_buffering off` for streaming, and `07-technology-qa-bank.md` covers SSE concepts, but `chat.py` currently returns buffered `ChatCompletionResponse` JSON (`stream=True` is unimplemented).
5. **Process-Local Circuit Breaker:** `CircuitBreaker` lives in worker RAM (`_global_router` singleton). Across $N$ Uvicorn workers/pods, breaker state drifts independently until externalized to Redis.
6. **No Active Prompt Sanitizer / RBAC Auth Layer Yet:** Earlier Day 0 PRD notes mentioned prompt-injection regexes and API-key authentication middleware, and `Q35` mentions `tiktoken(..., allowed_special=set())`, but current code only uses `X-API-Key` as a rate-limit bucket key and does not enforce API key auth or special-token/regex sanitization.
7. **Mocked vs. Live Benchmarks:** All 44 verified tests run against `FakeRedis` / `AsyncMock` / mocked `litellm.acompletion`. Target SLOs (e.g., `<25ms` cache hit, `<50ms` hybrid search at 500k chunks, `40–60%` token savings) are architectural targets / mocked assertions until load-tested against live datasets.

---

## 5. Individual Contribution & Ownership Framing

When asked *"What was your individual contribution?"*, avoid both exaggeration and helplessness:
- **What to own directly:** End-to-end architecture design, choosing the single-store PostgreSQL schema (`pgvector` + computed `tsvector`), implementing/verifying the atomic Redis Lua rate limiter, two-stage semantic cache, sequential asyncpg retrieval + RRF fusion, circuit-breaker/Tenacity failover pipeline, observability (`structlog` + Prometheus), test suite design, and Docker/Nginx deployment pipeline.
- **How to frame AI-assisted building honestly (if relevant, per `AI_BUILDER_TO_MASTER_BLUEPRINT.md`):** Modern senior engineers use AI tooling for scaffolding speed while owning architectural decisions, edge-case debugging (e.g., `asyncpg` socket collisions, `BaseHTTPMiddleware` ordering, `decode_responses=True` binary encoding, `EVALSHA` fallbacks), and failure-mode verification. Your ability to trace every line and explain every trade-off proves genuine ownership.

---
**Next:** [02-project-architecture.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/02-project-architecture.md)

