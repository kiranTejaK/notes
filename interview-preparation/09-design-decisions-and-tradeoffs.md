# 09. Design Decisions & Architectural Trade-Offs

Every architecture choice in SentinelRAG trades one complexity for another. Senior interviewers test whether you understand **both** sides—the advantages you gained, the downsides you accepted, and the concrete thresholds where you would switch approaches.

---

## Trade-Off 1: Single-Store PostgreSQL 16 (`pgvector` + `tsvector`) vs. Pinecone/Qdrant + Elasticsearch

| Dimension | SentinelRAG Choice: Single-Store PostgreSQL 16 | Alternative: Separate Postgres + Vector DB + Elasticsearch |
| :--- | :--- | :--- |
| **Consistency** | Single ACID transaction updates text, JSONB, vector, and `tsvector` together. | Dual-write risk; requires outbox pattern / Kafka / CDC reconciliation on updates/deletes. |
| **Operational Cost** | One database engine to back up, monitor, secure, and pool. | Three clusters to scale, patch, monitor, and pay network hops between. |
| **Metadata Filtering** | Native SQL `WHERE tenant_id = ...` + GIN JSONB alongside HNSW/GIN. | Cross-store ID joins or pre/post-filtering recall drop in external vector stores. |
| **Scale Ceiling** | Up to ~10M–50M vectors (HNSW graph must largely fit in RAM). | Scales to billions of vectors with distributed sharding and disk-backed quantization. |

- **Reproducible Interview Defense:**
  > *"For multi-tenant RAG up to millions of chunks, splitting relational metadata, vectors, and keyword search across three engines introduces dual-write consistency bugs and unnecessary infrastructure overhead. Keeping all three in one PostgreSQL 16 `document_chunks` table guarantees atomic updates and clean SQL tenant isolation. I'd decouple into Qdrant/Milvus only once vector RAM exceeds single-node limits (~10M–50M+ chunks) or ingestion write amplification degrades read p99."*

---

## Trade-Off 2: Vanilla Redis 7 Two-Tier Scan (SHA-256 + IEEE 754 Cosine) vs. Redis Stack `FT.SEARCH` vs. In-Process FAISS

- **Why not In-Process FAISS / Python Dict?** Fragmented hit rates across $N$ Gunicorn/Uvicorn workers ($1/N$ cache sharing), $N\times$ duplicated vector RAM across pods, and manual IPC invalidation. Paying ~1ms over Docker/VPC bridge to centralized Redis preserves 100% cache sharing and single-command tenant invalidation.
- **Why not Redis Stack `FT.SEARCH` Right Now?** `RediSearch` requires the `redis-stack-server` C module, which isn't available on every standard managed Redis tier (e.g., basic ElastiCache/vanilla `redis:7.2-alpine`).
- **Disadvantage Accepted:** Tier 2 executes `SMEMBERS` + pipelined `HGETALL` + an $O(N \cdot D)$ Python cosine loop. That stays under ~15–20ms when tenant active caches are small (hundreds to low thousands of entries bounded by 24h TTL + Tier 1 SHA-256 absorbing exact repeats), **but degrades linearly** past ~5,000–10,000 cached vectors per tenant.
- **When to Switch:** When active per-tenant cache cardinality exceeds ~5,000 entries, upgrade Tier 2 to RediSearch `FT.CREATE ... SCHEMA vector VECTOR HNSW` for sub-3ms C-level ANN search.

---

## Trade-Off 3: Atomic Redis Lua Token Bucket vs. Fixed Window `INCR` / `MULTI-EXEC` / Edge WAF Only

- **Why Lua over `GET/SET` or `MULTI/EXEC`?** `GET` then `SET` leaks burst quotas under microsecond concurrency across pods; `MULTI/EXEC` queues commands blindly and cannot compute conditional time-based token refills from read values inside one transaction.
- **Why Lua over Edge-Only (Nginx/Cloudflare)?** Edge WAFs excel at coarse IP/DDoS protection (and SentinelRAG retains Nginx at the perimeter), but lack application context (`X-API-Key`, dynamic per-tenant capacities, or future `tiktoken` token-weighted quotas) and typically return generic HTML `429` pages instead of structured RFC 7807 `application/problem+json` with exact `Retry-After`.
- **Disadvantages Accepted:** Redis becomes a dependency on every non-exempt request path (mitigated via fail-open catch + 500ms socket timeout), and Lua scripts must stay $O(1)$ so they never stall Redis's single thread.

---

## Trade-Off 4: Reciprocal Rank Fusion ($k=60$) vs. Linear Score Combination vs. Neural Cross-Encoder

- **Why not Linear Weighted Sum ($\alpha \cdot \text{dense} + (1-\alpha) \cdot \text{sparse}$)?** Cosine distance $[0, 2]$ and `ts_rank_cd` $[0, \infty)$ have incompatible scales and distributions; per-query min-max normalization amplifies weak-query noise (a lone `0.02` keyword score gets normalized to `1.0`).
- **Why not a Neural Cross-Encoder (`bge-reranker-large` / Cohere Rerank)?** Cross-encoders run full transformer cross-attention over `(query, chunk)` pairs, yielding top precision—but add 150–400ms CPU/GPU latency or paid external API calls per request.
- **Disadvantage of RRF:** Pure ordinal ranks treat Rank 1 the same whether cosine similarity was `0.98` or `0.55`, and miss deep token-level cross-attention interactions.
- **When to Switch:** Add a Cross-Encoder as a *second-stage reranker* solely on the top 5–10 RRF candidates for high-stakes legal/medical compliance search where +200ms latency is acceptable.

---

## Trade-Off 5: Custom `ResilientLLMRouter` (LiteLLM + Tenacity + `CircuitBreaker`) vs. LangChain/LlamaIndex vs. Envoy Mesh Only

- **Why avoid LangChain / LlamaIndex in the Gateway Core?** Heavy transitive dependencies, deep abstraction wrappers (`RunnableSequence`) that obscure production stack traces during outages, risk of hidden sync blocking inside wrappers, and minor-version churn. `LiteLLM` provides a thin, standardized OpenAI-schema adapter across Groq, OpenAI, and Mistral while SentinelRAG keeps explicit control over retries, circuit states, and `tiktoken` telemetry.
- **Why Python Circuit Breaker in addition to Envoy/Nginx?** Transport proxies see TCP drops and HTTP 5xx, but cannot parse LLM semantic error bodies, adapt provider-specific model parameters (`groq/llama-3.3` $\to$ `openai/gpt-4o-mini`), or recalculate `tiktoken` pricing on failover.
- **Disadvantage Accepted:** `_global_router` stores `CircuitBreaker` state in worker process RAM, causing multi-worker drift until synchronized via Redis Hash/PubSub.

---

## Trade-Off 6: FastAPI + Uvicorn (`uvloop`) vs. Flask/Django (WSGI) vs. Go / Rust

- **Why FastAPI over Flask/Django:** Gateway latency is dominated by waiting on external sockets (Redis, Postgres, LLMs). Cooperative async multiplexing handles thousands of open sockets per worker without allocating gigabytes of OS thread stacks, while Pydantic v2 (`pydantic-core` in Rust) delivers fast schema validation and OpenAPI docs.
- **Why Python over Go/Rust:** Native access to `tiktoken`, `litellm`, and rapid AI SDK evolution avoids maintaining brittle Go/Rust provider wrappers or CGO boundaries, while multi-process Uvicorn workers (`--workers`) bypass per-process GIL contention for I/O workloads.
- **When to Pivot to Go/Rust:** If the proxy shifts to 50k+ QPS raw L7 traffic, heavy token-stream parsing/redaction on every chunk, or strict `<256MB` edge footprints.

---

## Trade-Off 7: Current Implementation Shortcuts & Concrete Improvements

Being upfront about shortcuts in the current codebase proves senior engineering judgment:

| Current Implementation (`supplementary.md`) | Trade-Off / Limitation | Concrete Production Improvement |
| :--- | :--- | :--- |
| **`ChatCompletionRequest.query_vector` is caller-supplied** | Clients must pre-compute 1536d embeddings; omitting `query_vector` triggers `[0.0]*1536` dense search and skipped semantic caching. | Embed `query_text` server-side via `litellm.aembedding()` on Tier 1 exact-hash miss or when `query_vector is None`. |
| **`await cache_service.set(...)` runs inline in `chat.py`** | Adds 1 Redis pipeline RTT to every cache-miss response latency. | Dispatch cache write via FastAPI `BackgroundTasks` after response assembly. |
| **FastAPI `Depends(get_async_session)` spans the whole route** | Session stays checked out during the 1–3s upstream LLM call even after retrieval finishes! | Explicitly close/commit the read session right after `SearchRepository` returns before invoking `router_service.complete()`. |
| **`BaseHTTPMiddleware` in `rate_limiter.py`** | Extra Starlette streaming wrapper overhead and historical SSE cancellation quirks. | Rewrite as a pure ASGI middleware (`async def __call__(self, scope, receive, send)`). |
| **Native endianness `struct.pack(f"{n}f", ...)`** | Works when all workers share CPU architecture, breaks if x86 and ARM64 share one Redis. | Specify explicit little-endian `f"<{len(vector)}f"`. |
| **In-memory `CircuitBreaker` without `_probe_in_flight`** | Multi-worker drift + multiple coroutines can race during `HALF_OPEN`. | Add `_probe_in_flight` guard + `time.monotonic()`, and externalize state to Redis for multi-pod clusters. |

---
**Next:** [10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md)

