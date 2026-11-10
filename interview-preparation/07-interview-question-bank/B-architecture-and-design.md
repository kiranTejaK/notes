# Question Bank B: Architecture & System Design

---

## Q1: Explain the overall system architecture and trust boundaries.

### Layer 1 – Short Answer
> "SentinelRAG uses a four-tier containerized architecture: an edge Nginx tier (the only service exposing ports 80/443), a stateless non-root FastAPI/Uvicorn application tier, an isolated internal persistence tier (PostgreSQL 16 + pgvector and Redis 7.2 with zero host ports), and an outbound multi-provider LLM mesh (Groq, OpenAI, Mistral)."

### Layer 2 – Detailed Explanation & Component Communication
1. **Edge Tier (`nginx:1.27-alpine`):** Terminates TLS 1.2/1.3, enforces `50M` max request body, sets HSTS/security headers, disables `proxy_buffering` for streaming readiness, and maintains `keepalive 64` HTTP/1.1 connections to `api:8000` over `sentinel_prod_network`.
2. **Application Tier (`app/main.py`, UID 10001):** Runs `uvicorn` on `uvloop`. Uses a strict 5-tier acyclic module dependency graph (`core` $\to$ `db/models/repositories` $\to$ `services` $\to$ `api/middleware` $\to$ `main`) so higher controller tiers never leak into core primitives.
3. **Persistence Tier (Internal Docker Network Only):**
   - **Redis 7.2:** `512mb` max memory, `volatile-lru` eviction, AOF persistence, serving Lua rate limits (`EVALSHA`), semantic cache hashes (`24h` TTL), and atomic analytics counters.
   - **PostgreSQL 16 + `pgvector`:** Single `document_chunks` table holding text, JSONB (`GIN`), `Vector(1536)` (`HNSW`), and computed `TSVECTOR` (`GIN`).
4. **Outbound Egress:** `LiteLLM` async client calling Groq $\to$ OpenAI $\to$ Mistral over HTTPS with a `4.0s` per-call timeout.

### Layer 4 – Follow-up Questions
- *How are tenants isolated across all three storage/runtime tiers?*
  - **Postgres:** `tenant_id` (`String(64)`, B-tree indexed) is enforced in every `.where(DocumentChunk.tenant_id == tenant_id)` clause; UUIDv4 PKs prevent integer ID guessing.
  - **Redis Cache:** Keys use `cache:{tenant_id}:{sha256}` and `cache:{tenant_id}:keys`, structurally preventing cross-tenant matching or invalidation.
  - **Rate Limiter:** Buckets isolate per `ratelimit:key:{X-API-Key}` (or `ip:{host}`).
  - *Upgrade path:* Enforce PostgreSQL Row-Level Security (RLS) policies + Redis 6+ key-pattern ACLs (`~cache:tenant_x:*`).

---

## Q2: Why did you choose a single-store PostgreSQL 16 architecture instead of Pinecone/Qdrant + Elasticsearch?

### Layer 1 – Short Answer
> "Splitting relational metadata, dense vectors, and full-text search across three databases creates multi-database sprawl and the dual-write consistency trap. By storing text, JSONB metadata, `Vector(1536)` (HNSW), and a generated `TSVECTOR` (GIN) in one PostgreSQL 16 table, every ingestion or deletion is ACID-atomic without synchronization pipelines."

### Layer 2 – Detailed Explanation
- **Dual-Write Failure Mode:** In a split architecture, if a document update commits in Postgres and fails/times out in Pinecone or Elasticsearch, your RAG answers cite deleted or stale content unless you build an outbox + Kafka/Debezium CDC reconciliation pipeline.
- **Metadata Filtering Advantage:** Enterprise RAG queries almost always filter by `tenant_id` or JSONB attributes. External vector DBs suffer pre-filtering vs. post-filtering recall penalties; Postgres uses native B-tree + GIN + HNSW within one engine.
- **Operational Overhead:** One engine to back up (`pg_dump`), monitor, and pool connections for.

### Layer 3 – Project Connection
In `app/models/document.py`, `DocumentChunk` defines `embedding` (`Vector(1536)` with `m=16, ef_construction=64, vector_cosine_ops`) and `search_vector` (`Computed("to_tsvector('english', chunk_text)", persisted=True)` with `GIN`), populated atomically in `app/api/v1/documents.py` via bulk `INSERT ... RETURNING`.

### Layer 4 – Follow-up Questions
- *When would this single-store architecture stop scaling?*
  - When vector counts exceed ~10M–50M chunks (HNSW graph no longer fits comfortably in RAM) or write ingestion spikes cause Postgres WAL write-amplification and autovacuum pauses. Before migrating to Qdrant/Milvus, intermediate steps include `pgvector` `halfvec` (50% RAM cut), read replicas for `SearchRepository`, and PgBouncer.

---

## Q3: What architectural challenges did you solve in this design?

### Layer 1 – Short Answer
> "Three concrete challenges shaped the implementation: preventing `asyncpg` socket collisions during hybrid search, avoiding UTF-8 decoding crashes when caching float vectors in `redis.asyncio`, and keeping `HTTP 429` rate-limit rejections correlated and CORS-compliant."

### Layer 2 – Concrete Challenges & Resolutions
1. **`asyncpg` Half-Duplex Socket Collision:** Running `asyncio.gather(search_dense(), search_sparse())` on one SQLAlchemy `AsyncSession` raises `asyncpg.InterfaceError: another operation is in progress` because `asyncpg` uses one streaming TCP socket per session. Resolved by running the two sub-15ms queries sequentially in `SearchRepository.search_hybrid_candidates`.
2. **Redis `decode_responses=True` vs. Raw Binary Floats:** The Redis pool decodes responses as UTF-8 strings, which crashes on raw IEEE 754 binary bytes from `struct.pack`. Resolved by Base64-encoding the packed `6,144` bytes (`8,192` ASCII chars) in `app/services/cache.py`.
3. **Ghost Keys in Tenant Set Indexes:** TTL expiry deletes `cache:{tenant}:{hash}` hashes automatically, but does not remove members from `cache:{tenant}:keys`. Resolved via lazy `SREM` eviction during `get_or_lookup` scans.
4. **Middleware Wrapping Order:** Registering `correlation_id_and_timing_middleware` last in `app/main.py` ensures it runs *outside* CORS and `TokenBucketRateLimitMiddleware`, attaching `X-Request-Id` and timing telemetry even to `429` responses.

---

## Q4: How would SentinelRAG change if traffic scaled 100x (e.g., from 50 QPS to 5,000 QPS)?

### Layer 1 – Short Answer
> "At 5,000 QPS, three components become bottlenecks: Python-side linear vector scanning in the semantic cache, direct `asyncpg` connection counts against PostgreSQL, and process-local `CircuitBreaker` state drifting across workers. I would move cache vector matching into RediSearch HNSW (`FT.SEARCH`), introduce PgBouncer + Postgres read replicas, and externalize circuit breaker state to Redis."

### Layer 2 – Detailed Scaling Blueprint
1. **Redis Semantic Cache (`app/services/cache.py`):**
   - *Bottleneck:* `SMEMBERS` + `HGETALL` pipeline + Python cosine loop scales as $O(N \cdot D)$ per query. Beyond ~5,000 cached queries/tenant, network transfer and Python loop CPU saturate the event loop.
   - *Fix:* Switch to **RediSearch (`FT.CREATE` / `FT.SEARCH ... KNN`)** running C-based HNSW inside Redis, or shard cache lookups across Redis Cluster by `{tenant_id}` hash tags.
2. **PostgreSQL Connection Multiplication (`app/db/session.py`):**
   - *Bottleneck:* 20 API containers $\times$ 4 workers $\times$ `pool_size=20` (+10 overflow) = up to 2,400 Postgres backend processes ($\sim$12–24 GB RAM and spinlock thrashing).
   - *Fix:* Put **PgBouncer** (transaction pooling mode, disabling `asyncpg` statement cache via `statement_cache_size=0`) in front of Postgres, route `SearchRepository` reads to dedicated read replicas, and enforce `SET LOCAL statement_timeout = '100ms'` (degrading to sparse-only search on timeout).
3. **Distributed Circuit Breaker (`app/core/circuit_breaker.py`):**
   - *Bottleneck:* In-process `_global_router` means 10 pods must fail $10 \times 5 = 50$ times before all trip.
   - *Fix:* Store failure window and state in Redis Hash + Lua/PubSub so all replicas trip simultaneously.
4. **Gateway Throughput:**
   - Replace `BaseHTTPMiddleware` in `rate_limiter.py` with raw ASGI middleware, move `cache_service.set()` into `BackgroundTasks`, add server-side embedding generation with singleflight dedup, and enable `PROMETHEUS_MULTIPROC_DIR` across Uvicorn workers.

---
**Next:** [C-technology-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/C-technology-questions.md)

