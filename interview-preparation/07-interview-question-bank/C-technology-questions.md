# Question Bank C: Technology-Specific Deep Questions

Covers the core technologies actually used in SentinelRAG: FastAPI/ASGI, PostgreSQL/`pgvector`, Redis/Lua, LiteLLM/Tenacity/`tiktoken`, `structlog`/Prometheus, and Docker/Nginx/CI-CD/`pytest`.

---

## Q1: FastAPI & `asyncio` — What happens if a blocking call like `requests.get()` or `time.sleep()` runs inside an `async def` endpoint?

### Layer 1 – Short Answer
> "Because an ASGI worker's `asyncio` event loop executes on a single OS thread via cooperative multitasking, any blocking synchronous call inside `async def` freezes the entire thread. While it blocks, no other concurrent request on that worker—even `/healthz`—can make progress until the blocking call finishes."

### Layer 2 – Detailed Explanation
- In `async def`, coroutines yield control back to the event loop (`epoll`/`uvloop`) *only* at `await` on non-blocking awaitables. Synchronous I/O (`requests`, `psycopg2`, `time.sleep`) blocks the OS thread in kernel `recv()`/`nanosleep()`.
- By contrast, FastAPI executes standard `def` routes in an AnyIO worker thread pool (default 40 threads), avoiding event-loop starvation at the cost of thread-pool saturation under slow upstream calls.

### Layer 3 – Project Connection
SentinelRAG uses `async def` strictly with non-blocking drivers (`asyncpg`, `redis.asyncio`, `litellm.acompletion`/`httpx`), and `app/core/config.py` enforces `postgresql+asyncpg://` at startup so SQLAlchemy can never silently load synchronous `psycopg2`.

### Layer 4 – Follow-up Questions
- *How do you detect event loop lag in production?*
  - Schedule a periodic probe (`loop.call_later(0.01, probe)`) measuring `actual - scheduled` latency exported to Prometheus, or enable `PYTHONASYNCIODEBUG=1` in staging to flag callbacks exceeding `100ms`.
- *What if you must call a CPU-heavy or sync library inside an async route?*
  - Offload sync I/O via `await asyncio.to_thread(...)` or CPU-bound tasks (like large `tiktoken` batches or massive numpy arrays) to a `ProcessPoolExecutor`.

---

## Q2: `structlog` & `contextvars` — Why does `threading.local()` fail in FastAPI, and how does `contextvars` fix it?

### Layer 1 – Short Answer
> "`threading.local()` binds data to an OS thread, which breaks in `asyncio` because thousands of coroutines share the same thread and overwrite each other across `await` yields. Python's `contextvars` maintains task-local storage per `asyncio.Task` across `await` boundaries, letting `structlog` automatically tag every nested log with `request_id`."

### Layer 3 – Project Connection
In `app/main.py`, `correlation_id_and_timing_middleware` calls `structlog.contextvars.clear_contextvars()` before `bind_contextvars(request_id=..., route=..., method=...)`. Clearing at entry is vital so reused task frames never leak a previous request's `request_id`.

### Layer 4 – Follow-up Questions
- *What happens to `contextvars` when you spawn a background task via `asyncio.create_task()`?*
  - `asyncio.create_task()` copies the caller's `Context` at creation time, so the child task inherits `request_id` safely without mutating the parent's context.

---

## Q3: `pgvector` — Compare HNSW vs. IVFFlat, and explain `<=>`, `<->`, and `<#>` operators.

### Layer 1 – Short Answer
> "IVFFlat clusters vectors via k-means into Voronoi cells; it requires training on existing data and loses recall as distributions shift. HNSW builds a multi-layer proximity skip-list graph supporting incremental inserts with no training step, $>95\%$ recall, and $O(\log N)$ latency at higher RAM cost. For operators: `<=>` is cosine distance, `<->` is L2 Euclidean, and `<#>` is negative inner product."

### Layer 2 – Key Internals
- **Index Parameters (`app/models/document.py`):** `m=16` (bidirectional edges per node; higher = better recall + more RAM), `ef_construction=64` (build-time search queue depth), and runtime `SET hnsw.ef_search` (query-time recall vs. latency knob).
- **Operator Mismatch Trap:** The index specifies `postgresql_ops={"embedding": "vector_cosine_ops"}`, which only serves `<=>`. Querying `<->` silently falls back to a full `Seq Scan`.
- **Unit-Vector Tip:** When embeddings are unit-normalized ($\|u\|=1$), negative inner product `<#>` yields identical ordering to cosine distance `<=>` while skipping norm/sqrt math.

---

## Q4: PostgreSQL Full-Text Search — Why `GENERATED ALWAYS AS ... STORED`, `websearch_to_tsquery`, and `ts_rank_cd`?

### Layer 1 – Short Answer
> "Declaring `search_vector` as a persisted `GENERATED ALWAYS` column lets PostgreSQL automatically tokenize, remove stop words, and stem `chunk_text` in C on every insert/update without application code. `websearch_to_tsquery` safely parses raw user queries without syntax crashes, and `ts_rank_cd` scores cover density—rewarding terms appearing close together."

### Layer 3 – Project Connection
In `app/models/document.py`, `search_vector` uses SQLAlchemy `Computed("to_tsvector('english', chunk_text)", persisted=True)` with a `GIN` index. Because it is a generated column, `app/api/v1/documents.py` omits `search_vector` from bulk `INSERT` dictionaries—including it would fail with a PostgreSQL error (`can only be updated to DEFAULT`).

---

## Q5: Redis & Lua — Why use Lua `EVALSHA` and `redis.call('TIME')` instead of `GET/SET` or `MULTI/EXEC`?

### Layer 1 – Short Answer
> "`GET` then `SET` from Python creates check-then-act race conditions under concurrent requests, and `MULTI/EXEC` merely queues commands without conditional branching based on read values. A Redis Lua script executes atomically in Redis's single-threaded engine in one round-trip, and calling `redis.call('TIME')` inside Lua eliminates NTP clock skew across app containers."

### Layer 3 – Project Connection
`app/core/rate_limiter.py` caches the SHA-1 via `SCRIPT LOAD` and calls `EVALSHA` (falling back automatically to `EVAL` if `NoScriptError` occurs after a Redis restart or `SCRIPT FLUSH`). It also sets `EXPIRE` to `max(60, ceil((capacity / refill_rate) * 2))` so idle client buckets don't leak Redis memory.

### Layer 4 – Follow-up Questions
- *What is the downside of Redis Lua scripts?*
  - Long-running Lua scripts block the single-threaded Redis event loop for all clients (`lua-time-limit`, default 5s), and Lua does **not** roll back earlier writes if an error occurs mid-script. SentinelRAG's script executes $O(1)$ operations in microseconds.

---

## Q6: LiteLLM, Tenacity & `tiktoken` — Why Full Jitter, and how does `tiktoken` avoid token miscounting and injection?

### Layer 1 – Short Answer
> "Pure exponential backoff synchronizes client retries into periodic Thundering Herd spikes. Tenacity's Full Jitter samples `Uniform(0, min(max_wait, base * 2^attempt))`, spreading retries smoothly across time and filtering out permanent `400/401/403` errors. For cost tracking, `tiktoken` runs Rust-compiled BPE tokenization including ChatML message framing tokens (`+3` per message, `+3` primer) rather than inaccurate `chars/4` estimates."

### Layer 3 – Project Connection & Gap Check
- `app/services/resilience.py` uses `AsyncRetrying` with `wait_random_exponential` and `is_transient_error` (`429, 502, 503, 504`, timeouts, connection resets).
- `app/services/token_analytics.py` resolves per-model rates (`MODEL_PRICING`), falling back to `cl100k_base` for unfamiliar provider prefixes.
- *Security Gap to Note:* `tiktoken.encode(str(value))` should pass `allowed_special=set()` (or `disallowed_special=()`) so literal `<|endoftext|>` strings in user input are encoded as plain text instead of raising `ValueError` or risking special-token control injection.

---

## Q7: Prometheus, Docker & Nginx — Why `proxy_buffering off`, UID `10001`, and pre-deploy `alembic upgrade head`?

### Layer 1 – Short Answer
> "Nginx buffers upstream responses by default (32KB+), which breaks real-time LLM token streaming unless `proxy_buffering off` is set. Running Docker as `appuser` (`UID 10001`) without build compilers prevents container escapes from granting host root. And running `alembic upgrade head` as an isolated pre-deploy CI/CD job avoids DDL lock races and eliminates DDL permissions from runtime app containers."

### Layer 3 – Project Connection
- `Dockerfile`: Multi-stage `python:3.11-slim` with `uv sync --frozen --no-dev` + BuildKit cache mount in `builder`; only `libpq5` + `curl` + `.venv` + `UID 10001` in `runner`; exec-form `CMD ["uvicorn", ...]` so Uvicorn runs directly as PID 1 and traps `SIGTERM` cleanly.
- `nginx/conf.d/sentinel.conf`: TLS 1.2/1.3, `ssl_session_tickets off` (preserves Perfect Forward Secrecy), OCSP stapling, `keepalive 64`, `proxy_buffering off`.
- `app/core/telemetry.py`: Calibrates `http_request_duration_seconds` buckets `(0.005..10.0)` from 5ms cache hits to 10s LLM calls; keeps label cardinality bounded (`endpoint`, `status`, `result`, `model`, `tenant`).

---
**Next:** [D-implementation-and-operations.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/D-implementation-and-operations.md)

