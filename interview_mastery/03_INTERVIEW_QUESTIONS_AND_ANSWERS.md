# SentinelRAG: Comprehensive Topic-Wise Interview Question Bank

> **Target Audience:** Python Backend Developers (2–4 Years Experience) preparing for Senior / Mid-Senior Technical Interviews.  
> **Structure:** Questions organized across 15 core technical domains with production-grade answers, implementation references, likely follow-ups, and common misconceptions.  
> **Workspace Root:** [c:/Users/kiran/Desktop/sentinel-rag](file:///c:/Users/kiran/Desktop/sentinel-rag)

---

## Domain 1: Project Introduction & Architecture

### Q1.1: Can you introduce SentinelRAG, the core problem it solves, and its high-level architecture?
- **Priority:** `P1` (Must-Know)
- **Expected Answer:**
  > *"SentinelRAG is an enterprise-grade AI middleware gateway built with Python 3.12 and FastAPI. It sits between client applications and frontier LLM inference providers like Groq, OpenAI, and Mistral. 
  >
  > In production, direct client-to-LLM integrations face severe pain points: cold LLM generation latency of 1 to 3 seconds, runaway token costs from answering repetitive questions, vendor rate limits causing cascading outages, and lexical search failures when vector databases miss exact error codes or SKUs.
  >
  > SentinelRAG addresses this with a four-stage pipeline:
  > 1. A sub-20ms Redis Semantic Cache using exact SHA-256 matching and 1536-dimensional cosine distance to intercept repetitive queries.
  > 2. A single-store PostgreSQL 16 hybrid retrieval engine combining pgvector HNSW dense vectors and GIN-indexed tsvectors, fused via Reciprocal Rank Fusion ($k=60$).
  > 3. A resilient LiteLLM routing layer with a sliding-window Circuit Breaker that fails fast to secondary providers during Groq rate limits or outages.
  > 4. Distributed Redis Lua rate limiting, RFC 7807 error responses, and Prometheus telemetry to monitor latency and cumulative dollars saved."*
- **Implementation Reference:** [app/main.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/main.py#L61-L102), [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py#L60-L330).
- **Likely Follow-Up:** *"Why build a custom gateway instead of using an off-the-shelf product like Portkey or LiteLLM Proxy?"*
- **Common Mistake:** Describing it merely as a "wrapper around OpenAI" without highlighting the semantic cache, hybrid search math, or circuit breaker state machine.

---

### Q1.2: Why did you choose a single-store PostgreSQL architecture over a dedicated vector database like Pinecone or Qdrant?
- **Priority:** `P1`
- **Expected Answer:**
  > *"Dedicated vector databases introduce dual-system operational sprawl: two separate data stores to provision, monitor, back up, and secure. Most critically, they create dual-write consistency problems: if a document is updated or deleted in the relational database, there is an eventual consistency lag before the vector database reflects the change, leading to stale or orphaned embeddings.
  >
  > In SentinelRAG, PostgreSQL 16 with `pgvector` allows us to store the raw chunk text, JSONB metadata, a 1536-dimensional dense embedding (indexed via HNSW), and an automatically generated `tsvector` column (indexed via GIN) in the exact same table row. This gives us ACID transactions across metadata and vectors, zero network hops between stores, and atomic batch insertions. For our scale (< 5 million chunks), HNSW in pgvector delivers sub-50ms query latency, matching dedicated vector stores while drastically simplifying infrastructure."*
- **Implementation Reference:** [app/models/document.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/models/document.py#L16-L84), [app/repositories/search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L34-L152).
- **Likely Follow-Up:** *"At what scale would PostgreSQL and pgvector break down, forcing a migration to a dedicated vector database?"*
- **Common Mistake:** Claiming PostgreSQL is always better than Pinecone without acknowledging RAM constraints for HNSW graphs at 100M+ vector scales.

---

## Domain 2: End-to-End Request Flows

### Q2.1: Trace the lifecycle of a `POST /api/v1/chat/completions` request from the network socket to the response.
- **Priority:** `P1`
- **Expected Answer:**
  > *"First, the TCP connection hits Nginx, terminates TLS, and forwards to Uvicorn running our FastAPI application.
  > 1. The request enters `TokenBucketRateLimitMiddleware`. It extracts the `X-API-Key`, calls an atomic Redis Lua script, refills tokens based on server time, and decrements 1 token. If empty, it returns HTTP 429 immediately.
  > 2. The correlation middleware generates an `X-Request-Id` and binds it to `structlog` ContextVars for structured logging.
  > 3. Inside `chat_completions`, Stage 1 checks `RedisSemanticCache`. If an SHA-256 hash or vector cosine distance ($\le 0.08$) matches, it calculates saved tokens via `tiktoken` and returns an assistant response in $< 20$ms.
  > 4. On a cache miss, Stage 2 triggers `SearchRepository`. It runs two sequential queries against PostgreSQL 16: pgvector cosine distance (`<=>`) and full-text `ts_rank_cd`. The top 50 candidates from each are merged using Reciprocal Rank Fusion ($k=60$), and the top 5 chunks are injected into the system prompt.
  > 5. Stage 3 calls `ResilientLLMRouter`. If the Circuit Breaker is `CLOSED`, it calls Groq Llama-3.3-70B wrapped in a Tenacity exponential backoff retry. If Groq fails repeatedly, the breaker trips to `OPEN`, and the request falls back to OpenAI `gpt-4o-mini`.
  > 6. Stage 4 writes the generated response to Redis with a 24-hour TTL, increments Prometheus token and spend metrics, and returns the OpenAI-compatible JSON payload."*
- **Implementation Reference:** [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py#L60-L330).
- **Likely Follow-Up:** *"Why do you run the dense and sparse queries sequentially rather than with `asyncio.gather`?"*
- **Common Mistake:** Forgetting the rate limiting or correlation middleware stages, or claiming the cache and DB are queried concurrently.

---

## Domain 3: Python & ASGI Concurrency Internals

### Q3.1: How does Python's AsyncIO event loop work, and why must blocking operations never be called inside async route handlers?
- **Priority:** `P1`
- **Expected Answer:**
  > *"Python's `asyncio` is a single-threaded cooperative multitasking event loop. It maintains a ready queue of tasks and an I/O polling selector (such as `epoll` on Linux or `kqueue` on macOS). When a coroutine reaches an `await` on a non-blocking I/O operation (like `asyncpg` network reads or `redis.asyncio` commands), it yields execution control back to the event loop, which immediately schedules another runnable task.
  >
  > If a developer executes a blocking synchronous call—such as `time.sleep()`, synchronous `requests.get()`, or heavy CPU-bound serialization—inside an `async def` function, that single thread blocks entirely. The event loop cannot tick, no other coroutines can resume, and all concurrent user requests on that Uvicorn worker freeze, causing latency spikes and connection timeouts across the entire service."*
- **Implementation Reference:** Used throughout; non-blocking drivers `asyncpg` and `redis.asyncio` are used exclusively; monotonic timing uses `time.perf_counter()`.
- **Likely Follow-Up:** *"If you had a CPU-intensive operation like computing a high-dimensional vector similarity across 50,000 items in Python, how would you prevent it from blocking the event loop?"*
- **Common Mistake:** Thinking `async def` creates OS threads automatically or that `await` makes synchronous code asynchronous.

---

### Q3.2: How does `structlog` ContextVars work in an asynchronous ASGI application, and why is standard `threading.local` broken in async Python?
- **Priority:** `P2`
- **Expected Answer:**
  > *"`threading.local()` associates state with a specific operating system thread. In an ASGI framework like FastAPI, dozens of interleaved coroutines run concurrently on the exact same OS thread within the event loop. If one coroutine sets `thread_local.request_id = 'req_1'` and yields at an `await`, another coroutine could overwrite it with `thread_local.request_id = 'req_2'`. When the first coroutine resumes, its context is corrupted.
  >
  > Python 3.7+ introduced `contextvars`. A `ContextVar` maintains context local to the current `asyncio.Task`. When a new task is spawned or a coroutine yields and resumes, the event loop automatically manages and restores the active task's Context. In SentinelRAG, [app/core/logging.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/logging.py) uses `structlog.contextvars.bind_contextvars(request_id=...)`, ensuring every log message throughout nested service calls is tagged with the correct request ID without passing it as an argument."*
- **Implementation Reference:** [app/main.py#L112-L117](file:///c:/Users/kiran/Desktop/sentinel-rag/app/main.py#L112-L117), [app/core/logging.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/logging.py).
- **Likely Follow-Up:** *"What happens to contextvars when you spawn a background task with `asyncio.create_task`?"*
- **Common Mistake:** Believing `contextvars` is shared globally or that it causes memory leaks if not manually reset.

---

## Domain 4: FastAPI, Routing, Dependency Injection & Middleware

### Q4.1: How does FastAPI's Dependency Injection (`Depends`) system handle connection lifecycle and resource cleanup?
- **Priority:** `P1`
- **Expected Answer:**
  > *"FastAPI's dependency injection system builds an internal dependency graph per request. When a dependency is implemented using a Python generator function (`yield`), FastAPI manages the lifecycle in two phases:
  > 1. Everything before the `yield` runs prior to entering the route handler (setup).
  > 2. The yielded object is injected into the route parameters.
  > 3. After the route returns or raises an unhandled exception, FastAPI returns to the dependency generator and executes the code after the `yield` inside a `finally` block (cleanup).
  >
  > In SentinelRAG, [get_async_session](file:///c:/Users/kiran/Desktop/sentinel-rag/app/db/session.py#L41-L56) yields an `AsyncSession`. If an exception occurs during the route execution, it catches the error, calls `await session.rollback()`, and then ensures `await session.close()` in the `finally` block. This guarantees database connections are never leaked back to the pool in a dirty transaction state."*
- **Implementation Reference:** [app/db/session.py#L41-L56](file:///c:/Users/kiran/Desktop/sentinel-rag/app/db/session.py#L41-L56), [app/core/database.py#L59-L71](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/database.py#L59-L71).
- **Likely Follow-Up:** *"What is the difference between `async def` dependencies and standard `def` dependencies in FastAPI?"*
- **Common Mistake:** Confusing `Depends` with simple function calls and failing to explain how `yield` guarantees cleanup.

---

### Q4.2: Why does SentinelRAG implement RFC 7807 Problem Details for HTTP APIs?
- **Priority:** `P2`
- **Expected Answer:**
  > *"Standard HTTP status codes (like 400 or 429) do not convey sufficient machine-readable context. Default error formats often vary across endpoints—one returning `{"error": "string"}` and another returning `{"detail": [...]}`.
  >
  > RFC 7807 defines a standardized JSON schema (`application/problem+json`) with five core attributes: `type` (a URI reference identifying the error type), `title` (human-readable summary), `status` (HTTP code), `detail` (specific explanation of this occurrence), and `instance` (the request URI path). In SentinelRAG, [ProblemDetailException](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/exceptions.py#L9-L26) and [problem_detail_exception_handler](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/exceptions.py#L51-L82) standardize all rate limit violations, database outages, and schema validation errors, appending the unique `request_id` for distributed tracing."*
- **Implementation Reference:** [app/core/exceptions.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/exceptions.py).
- **Likely Follow-Up:** *"How does your rate limiter incorporate RFC 7807 during HTTP 429 rejections?"*
- **Common Mistake:** Thinking RFC 7807 is just an internal company convention rather than an IETF standard.

---

## Domain 5: Database Design, SQL, Transactions & pgvector Internals

### Q5.1: Explain the mathematical and algorithmic differences between HNSW and IVFFlat index types in pgvector.
- **Priority:** `P1`
- **Expected Answer:**
  > *"Both are Approximate Nearest Neighbor (ANN) vector indexing algorithms, but they use completely different data structures:
  > 1. **IVFFlat (Inverted File Flat):** Partitions vector space into Voronoi cells using k-means clustering. At query time, it probes the nearest $N$ centroids (`probes`) and scans vectors within those lists. IVFFlat requires an explicit training step on representative data (`CREATE INDEX ... WITH (lists = 1000)`). If data is inserted after training that alters the distribution, recall drops significantly. Build time is fast and RAM usage is low, but search latency and recall are mediocre.
  > 2. **HNSW (Hierarchical Navigable Small World):** Builds a multi-layer graph where lower layers contain dense connections between nearby vectors and higher layers contain long-range 'skip' connections (like a probabilistic skip-list). Queries start at the top layer, greedily routing to nearest neighbors before descending to fine-tune search at Layer 0.
  >
  > In SentinelRAG, we chose HNSW (`m=16`, `ef_construction=64`). It requires more RAM to store graph adjacency lists, but it offers $> 98\%$ recall, sub-50ms search times, and does not require a pre-training step—it supports incremental real-time insertions seamlessly."*
- **Implementation Reference:** [app/models/document.py#L71-L78](file:///c:/Users/kiran/Desktop/sentinel-rag/app/models/document.py#L71-L78).
- **Likely Follow-Up:** *"What do the parameters `m` and `ef_construction` control in HNSW?"*
- **Common Mistake:** Thinking pgvector uses exact K-Nearest Neighbor (KNN) brute-force search by default.

---

### Q5.2: In `SearchRepository`, why did you execute the dense vector query and sparse full-text query sequentially rather than concurrently using `asyncio.gather`?
- **Priority:** `P1` (Critical Code-Aware Question)
- **Expected Answer:**
  > *"This was an intentional architectural safeguard driven by `asyncpg` internals. 
  >
  > In SQLAlchemy 2.0 with the `asyncpg` driver, an `AsyncSession` borrows a single underlying physical database connection from the connection pool. The PostgreSQL wire protocol over a single TCP socket is strictly half-duplex and request-response driven; a single connection cannot pipeline multiple active query executions concurrently.
  >
  > If we had executed `await asyncio.gather(self.search_dense(...), self.search_sparse(...))` on the same `self._session`, both coroutines would attempt to write wire protocol packets and read socket buffers simultaneously, triggering `asyncpg.exceptions.InterfaceError: cannot perform operation: another operation is in progress`.
  >
  > To execute them concurrently, we would have had to check out two separate database sessions (consuming two connection pool slots for a single HTTP request). Because each query executes in $< 15$ms against its respective HNSW and GIN indexes, running them sequentially on a single session avoids connection pool exhaustion while guaranteeing socket safety."*
- **Implementation Reference:** [app/repositories/search.py#L140-L150](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L140-L150).
- **Likely Follow-Up:** *"If query volume grew to 10,000 QPS, how would you optimize this hybrid search step?"*
- **Common Mistake:** Suggesting `asyncio.gather` on the same session as an easy performance optimization without realizing it breaks asyncpg.

---

### Q5.3: Why does SentinelRAG configure `expire_on_commit=False` and `pool_pre_ping=True` in its SQLAlchemy session factory?
- **Priority:** `P2`
- **Expected Answer:**
  > *"1. `expire_on_commit=False`: By default, SQLAlchemy expires all model attributes after a transaction commits (`session.commit()`). In synchronous code, accessing an attribute later transparently triggers a lazy-load query. In asynchronous code (`asyncio`), transparent lazy-loading cannot happen because attribute access (`chunk.document_name`) is synchronous and cannot `await` an I/O query, triggering `GreenletSpawn` or `MissingGreenlet` exceptions. Setting `expire_on_commit=False` keeps the in-memory object attributes populated after commit.
  >
  > 2. `pool_pre_ping=True`: In production, firewalls, cloud load balancers, or PostgreSQL restart events can silently drop idle TCP sockets. Without pre-ping, when an ASGI worker checks out a connection from the pool, it attempts to write to a dead socket, causing an uncaught `ConnectionResetError`. Setting `pool_pre_ping=True` causes SQLAlchemy to execute a lightweight `SELECT 1` ping upon checkout; if the socket is dead, it recycles the connection transparently."*
- **Implementation Reference:** [app/db/session.py#L28-L38](file:///c:/Users/kiran/Desktop/sentinel-rag/app/db/session.py#L28-L38).
- **Likely Follow-Up:** *"What is the latency penalty of `pool_pre_ping=True`, and is it worth it?"*
- **Common Mistake:** Not knowing why lazy-loading fails in async SQLAlchemy.

---

## Domain 6: Distributed Caching & Redis Internals

### Q6.1: Why did you write an atomic Redis Lua script for rate limiting instead of using Redis `INCR` or Python-side logic?
- **Priority:** `P1`
- **Expected Answer:**
  > *"Standard Redis commands (`INCR`, `GET`, `SET`) or application-side math suffer from race conditions under concurrent burst traffic. If two requests arrive at two separate ASGI worker pods within the same microsecond, both read `tokens = 1`, compute elapsed refill, subtract 1, and write back `tokens = 0`. Both requests are granted access, permitting traffic that violates the rate limit quota.
  >
  > Distributed locks (`Redlock`) solve race conditions but introduce substantial network latency hops for every incoming HTTP request.
  >
  > A Redis Lua script executes atomically within Redis's single-threaded event loop. No other Redis command can execute while the script runs. Our Lua script queries the Redis server time (`redis.call('TIME')`), calculates elapsed milliseconds since `last_updated`, adds replenished tokens, compares against requested tokens, writes the new balance via `HSET`, and updates key expiration—all in a single non-blocking atomic operation with zero distributed lock overhead."*
- **Implementation Reference:** [app/core/rate_limiter.py#L17-L69](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L17-L69).
- **Likely Follow-Up:** *"How does `script_load` and `evalsha` improve performance over plain `eval`?"*
- **Common Mistake:** Suggesting Python-level `asyncio.Lock` for rate limiting across multiple server instances.

---

### Q6.2: How does SentinelRAG's semantic cache serialize embeddings, and why not store them as JSON lists?
- **Priority:** `P2`
- **Expected Answer:**
  > *"A 1536-dimensional embedding vector is an array of 1536 32-bit floating-point numbers. If serialized as a JSON string (`"[0.0123456, -0.0987654, ...]"`), each float takes ~8 characters plus commas and whitespace, resulting in ~14 KB per vector. Furthermore, `json.loads()` must parse 1536 ASCII strings into Python floats, which is slow and CPU-intensive.
  >
  > In [app/services/cache.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L40-L53), we use Python's C-accelerated `struct` module: `struct.pack(f"{len(vector)}f", *vector)`. This packs the 1536 floats directly into IEEE 754 32-bit binary representation, taking exactly $1536 \times 4 = 6,144$ bytes. We then base64-encode it to ~8,192 ASCII characters. This cuts memory footprint by ~45% and allows deserialization via `struct.unpack` in microseconds."*
- **Implementation Reference:** [app/services/cache.py#L40-L53](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L40-L53).
- **Likely Follow-Up:** *"How would you transition this semantic cache to native RediSearch Vector Similarity (VSS)?"*
- **Common Mistake:** Believing JSON serialization has negligible overhead on high-throughput vector lookups.

---

## Domain 7: Security, Authentication & Multi-Tenancy

### Q7.1: How does SentinelRAG achieve data isolation between different corporate tenants?
- **Priority:** `P1`
- **Expected Answer:**
  > *"SentinelRAG enforces logical multi-tenant isolation across all storage and caching tiers using a strict `tenant_id` partitioning strategy:
  > 1. **Database Tier:** Every row in `document_chunks` has a non-nullable `tenant_id` column indexed by B-Tree. All queries in `SearchRepository` strictly bind `.where(DocumentChunk.tenant_id == tenant_id)`.
  > 2. **Cache Tier:** Keys are strictly namespaced: `cache:{tenant_id}:{query_hash}`. A tenant cannot inspect or hit another tenant's cached responses, preventing cross-tenant data leakage. Tenant tracking sets (`cache:{tenant_id}:keys`) ensure tenant invalidation only purges that tenant's entries.
  > 3. **Rate Limiting Tier:** Quotas are enforced on `ratelimit:key:{api_key}` or `ratelimit:ip:{client_ip}`, preventing one tenant from starving the system's capacity."*
- **Implementation Reference:** [app/models/document.py#L33-L38](file:///c:/Users/kiran/Desktop/sentinel-rag/app/models/document.py#L33-L38), [app/services/cache.py#L94-L101](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L94-L101).
- **Likely Follow-Up:** *"How would you upgrade logical tenant isolation to row-level security (RLS) in PostgreSQL?"*
- **Common Mistake:** Suggesting a separate database or schema per tenant without explaining the operational burden at 10,000 tenants.

---

### Q7.2: What container security best practices are implemented in SentinelRAG's Dockerfile?
- **Priority:** `P2`
- **Expected Answer:**
  > *"In [Dockerfile](file:///c:/Users/kiran/Desktop/sentinel-rag/Dockerfile):
  > 1. **Multi-Stage Build:** We separate the builder stage (containing build tools, gcc, headers) from the final runtime stage, keeping the attack surface and image size minimal.
  > 2. **Non-Root Execution:** We create a dedicated non-root user and group (`appuser:appgroup` UID/GID 10001) and switch to `USER appuser`. If an attacker achieves Remote Code Execution (RCE) via an unpatched vulnerability, they cannot modify host volumes, install rootkits, or bind privileged ports (< 1024).
  > 3. **No Secrets in Image Layers:** Environment variables containing API keys and database passwords are never copied into the Dockerfile or baked into image layers; they are injected at runtime via environment variables or Kubernetes Secrets.
  > 4. **Minimal Base Image:** Uses `python:3.12-slim-bookworm` rather than full Debian images, reducing known CVEs."*
- **Implementation Reference:** [Dockerfile](file:///c:/Users/kiran/Desktop/sentinel-rag/Dockerfile).
- **Likely Follow-Up:** *"Why should you never use `latest` tags in base Docker images for production?"*
- **Common Mistake:** Running containers as `root` or hardcoding test secrets into Dockerfile `ENV` directives.

---

## Domain 8: External Integrations, Networking & Reliability

### Q8.1: Explain the mathematical mechanics of Exponential Backoff with Full Jitter. Why is plain exponential backoff insufficient?
- **Priority:** `P1`
- **Expected Answer:**
  > *"Plain exponential backoff calculates delay as $T_{\text{wait}} = \text{base} \times 2^{\text{attempt}}$. For example, with base 0.5s: 0.5s, 1.0s, 2.0s, 4.0s.
  > 
  > The problem with plain exponential backoff is that if a provider experiences a momentary outage, hundreds of concurrent requests fail at the exact same instant ($t=0$). All 100 requests sleep for exactly 0.5s, wake up simultaneously at $t=0.5\text{s}$, and hammer the recovering provider together. They fail again, sleep for exactly 1.0s, and strike again at $t=1.5\text{s}$. This is the **Thundering Herd** problem.
  >
  > In SentinelRAG, [app/services/resilience.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/resilience.py#L87-L88) implements Full Jitter:
  > $$T_{\text{sleep}} \sim \text{Uniform}\left(0, \min(T_{\text{max}}, \text{multiplier} \cdot 2^{\text{attempt}})\right)$$
  > By selecting a uniformly random sleep time between 0 and the exponential cap, the retries from those 100 clients are spread evenly across the time spectrum, flattening traffic spikes and allowing the provider to recover."*
- **Implementation Reference:** [app/services/resilience.py#L78-L109](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/resilience.py#L78-L109).
- **Likely Follow-Up:** *"How does Tenacity's `wait_random_exponential` implement this under the hood?"*
- **Common Mistake:** Confusing Jitter with random retries without an exponential upper bound.

---

### Q8.2: How does the Circuit Breaker state machine prevent cascading latency during third-party LLM outages?
- **Priority:** `P1`
- **Expected Answer:**
  > *"When an upstream service (like Groq) goes down, requests don't fail immediately; they typically hang until a client-side timeout occurs (e.g., 4 to 30 seconds). If 100 concurrent requests arrive, 100 worker coroutines hang waiting for network sockets to drop.
  >
  > SentinelRAG's [CircuitBreaker](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py#L41-L184) acts as a state machine:
  > - In `CLOSED`, it monitors errors. When Groq fails 5 consecutive times, the breaker trips to `OPEN`.
  > - In `OPEN`, [can_execute()](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py#L85-L106) immediately returns `False`. The router skips Groq entirely (0ms network delay) and dispatches straight to OpenAI `gpt-4o-mini`. Gateway latency stays low, and Groq is protected from further load.
  > - After 60 seconds (recovery timeout), the breaker transitions to `HALF_OPEN` and permits a single canary probe to Groq. If that probe succeeds, it resets to `CLOSED`; if it fails, it trips back to `OPEN` for another 60 seconds."*
- **Implementation Reference:** [app/core/circuit_breaker.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py), [app/services/router.py#L108-L127](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py#L108-L127).
- **Likely Follow-Up:** *"What would happen if the fallback provider also fails?"*
- **Common Mistake:** Thinking the circuit breaker catches errors, rather than proactively preventing calls when in the OPEN state.

---

## Domain 9: Observability, Logging, Metrics & Testing

### Q9.1: How does SentinelRAG correlate logs across asynchronous boundaries?
- **Priority:** `P1`
- **Expected Answer:**
  > *"In an asynchronous application, log lines from different requests interleave on stdout within milliseconds. Standard grepping by timestamp is useless.
  >
  > SentinelRAG solves this in [correlation_id_and_timing_middleware](file:///c:/Users/kiran/Desktop/sentinel-rag/app/main.py#L105-L141):
  > 1. It extracts `X-Request-Id` from incoming headers or generates a unique ID: `f"req_{uuid.uuid4().hex[:16]}"`.
  > 2. It sets `request.state.request_id = request_id`.
  > 3. It clears and binds this ID, along with `route` and `method`, to `structlog.contextvars`.
  > 4. As execution moves through service layers, any call to `logger.info(...)` automatically includes the bound `request_id`, `tenant_id`, and timing metrics in its structured JSON output.
  > 5. Finally, the middleware injects `X-Request-Id` into the HTTP response headers, allowing the client to provide the exact correlation ID for debugging."*
- **Implementation Reference:** [app/main.py#L105-L141](file:///c:/Users/kiran/Desktop/sentinel-rag/app/main.py#L105-L141), [app/core/logging.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/logging.py).
- **Likely Follow-Up:** *"Why output logs in JSON format rather than human-readable text in production?"*
- **Common Mistake:** Manually passing `request_id` as a parameter to every service function.

---

### Q9.2: How do you mock asynchronous dependencies like PostgreSQL sessions and Redis in `pytest`?
- **Priority:** `P2`
- **Expected Answer:**
  > *"We use `pytest-asyncio` combined with FastAPI's `app.dependency_overrides`.
  >
  > For database testing (e.g., [tests/test_hybrid_search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_hybrid_search.py#L18-L24)), we create an `AsyncMock` representing the SQLAlchemy session:
  > ```python
  > mock_session = AsyncMock()
  > async def override_get_async_session():
  >     yield mock_session
  > app.dependency_overrides[get_async_session] = override_get_async_session
  > ```
  > We then use `unittest.mock.patch` with `new_callable=AsyncMock` on repository or service methods to return deterministic fake domain models without requiring a running PostgreSQL database. Finally, we execute requests using `httpx.AsyncClient` with `ASGITransport(app=app)` and assert response JSON and status codes. In the `finally` block, we call `app.dependency_overrides.clear()` to prevent cross-test state leakage."*
- **Implementation Reference:** [tests/test_hybrid_search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_hybrid_search.py), [tests/test_chat_completions.py](file:///c:/Users/kiran/Desktop/sentinel-rag/tests/test_chat_completions.py).
- **Likely Follow-Up:** *"When would you choose unit tests with mocks versus integration tests with testcontainers?"*
- **Common Mistake:** Using synchronous `MagicMock` for async methods, causing `coroutine was never awaited` runtime warnings.

---

## Domain 10: Performance, Scalability & System Design

### Q10.1: If SentinelRAG's traffic grew from 50 QPS to 5,000 QPS, what would be the primary bottlenecks and how would you scale the system?
- **Priority:** `P1`
- **Expected Answer:**
  > *"At 5,000 QPS, the architecture would encounter three bottlenecks:
  > 1. **Redis Semantic Cache Vector Scan:** Currently, `get_or_lookup` pipelines an `HGETALL` across all tenant candidate keys and computes cosine distance in Python. At 10,000 cached vectors, this loop takes ~50ms.
  >    *Scaling Solution:* Migrate from pipelined Python scanning to **RediSearch (Redis Vector Similarity Index)** using native C-based HNSW indexing (`FT.SEARCH ... KNN`). This keeps vector similarity evaluations inside Redis under 3ms.
  > 2. **PostgreSQL Connection Pool Exhaustion:** With 20 pool connections per pod and multiple pods, PostgreSQL max connections (e.g., 200) would saturate.
  >    *Scaling Solution:* Deploy **PgBouncer** in transaction pooling mode between SentinelRAG and PostgreSQL 16. Furthermore, create read-replicas for `SearchRepository` queries since RAG candidate retrieval is read-only.
  > 3. **Process-Local Circuit Breaker Drift:** Each Uvicorn worker maintains its own failure counter in memory.
  >    *Scaling Solution:* Externalize circuit breaker state to Redis using atomic hashes and Pub/Sub so that when Groq trips, all 50 gateway pods immediately enter `OPEN` simultaneously."*
- **Implementation Reference:** [app/services/cache.py#L163-L190](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L163-L190), [app/db/session.py#L22-L30](file:///c:/Users/kiran/Desktop/sentinel-rag/app/db/session.py#L22-L30).
- **Likely Follow-Up:** *"Why can't PgBouncer be used with prepared statements in transaction pooling mode?"*
- **Common Mistake:** Vaguely answering "just add more Kubernetes pods" without identifying database socket limits and cache loop bottlenecks.

---

## Domain 11: AI/LLM Domain Concepts (Applied to Backend)

### Q11.1: Why does SentinelRAG use Cosine Distance rather than Euclidean (L2) or Inner Product (Dot Product) for vector similarity?
- **Priority:** `P2`
- **Expected Answer:**
  > *"Cosine distance measures the angle between two vectors, regardless of their magnitude:
  > $$\text{Cosine Similarity} = \frac{A \cdot B}{\|A\| \|B\|}, \quad \text{Cosine Distance} = 1.0 - \text{Cosine Similarity}$$
  > Euclidean (L2) distance measures the straight-line distance between vector endpoints. If one document chunk is long and contains frequent terms, its vector magnitude will be larger than a short chunk, skewing L2 distance even if their conceptual topic is identical.
  >
  > Most modern embedding models (such as OpenAI's `text-embedding-3-small` used in SentinelRAG) produce normalized unit vectors ($\|A\| = 1$). For unit vectors, Cosine Distance and Euclidean Distance are monotonically related, and Dot Product equals Cosine Similarity. Using `vector_cosine_ops` (`<=>`) in pgvector is standard, robust against unnormalized vectors, and directly maps to our semantic cache decision boundary ($\le 0.08$ distance)."*
- **Implementation Reference:** [app/models/document.py#L76](file:///c:/Users/kiran/Desktop/sentinel-rag/app/models/document.py#L76), [app/services/cache.py#L55-L76](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L55-L76).
- **Likely Follow-Up:** *"If vectors are pre-normalized, how can you optimize pgvector index search speed?"*
- **Common Mistake:** Confusing Cosine Distance (where 0.0 is identical) with Cosine Similarity (where 1.0 is identical).

---

### Q11.2: How does Reciprocal Rank Fusion (RRF) eliminate the "Score Calibration" problem in hybrid retrieval?
- **Priority:** `P2`
- **Expected Answer:**
  > *"When combining vector search and lexical search, their raw scores operate on completely incompatible scales:
  > - Dense Vector Cosine Distance is in $[0, 2]$, where smaller is better.
  > - Full-text BM25 / `ts_rank_cd` is an unbounded float $[0, \infty)$, where larger is better.
  >
  > If you attempt to combine them linearly: $\text{Score} = \alpha \cdot \text{VectorScore} + (1-\alpha) \cdot \text{LexicalScore}$, you must normalize the scores. However, min-max normalization depends entirely on the minimum and maximum scores of the *current result set*, which changes wildly between queries. If a query has no strong keyword match, the top keyword score might be $0.05$, but min-max will scale it to $1.0$, falsely inflating its relevance.
  >
  > RRF eliminates score calibration completely by ignoring raw scores and operating purely on ordinal ranks:
  > $$\text{RRF\_Score}(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
  > Whether a document was retrieved by a dense vector or a sparse keyword search, its contribution is governed strictly by whether it was ranked 1st, 2nd, or 10th."*
- **Implementation Reference:** [app/services/rrf.py#L69-L141](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/rrf.py#L69-L141).
- **Likely Follow-Up:** *"What are the limitations of RRF compared to a trained Cross-Encoder re-ranker?"*
- **Common Mistake:** Suggesting that converting cosine distance to `1 - distance` allows simple addition with BM25 scores.
