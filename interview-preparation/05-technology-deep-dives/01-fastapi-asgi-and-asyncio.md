# Technology Deep Dive 1: FastAPI, ASGI, Uvicorn (`uvloop`) & Pydantic v2

## 1. Fundamental Concepts & Internals

### WSGI vs. ASGI & `uvloop`
- **WSGI (PEP 3333):** Synchronous interface `application(environ, start_response)`. One OS thread/process blocks for the entire duration of an upstream call. At 1,000 slow LLM connections (2s each), you need 1,000 threads (~2–8 GB stack RAM + kernel scheduling thrash).
- **ASGI (`async def app(scope, receive, send)`):** Event-driven interface running on `asyncio`. Uvicorn translates raw TCP bytes into `scope` (metadata dict), `receive()` (awaitable request body stream), and `send()` (awaitable response chunk emitter).
- **`uvloop`:** Replaces Python's default `SelectorEventLoop` with a Cython wrapper over **`libuv`** (the C async I/O engine used by Node.js). It eliminates Python bytecode dispatch on socket readiness events, polling the kernel via `epoll` (Linux) / `kqueue` (macOS) with `epoll_wait()`, sleeping at 0% CPU when idle and waking up in $O(k)$ for $k$ ready sockets.

### `async def` vs. `def` Route Handlers
| Route Signature | Where It Runs | Concurrency Hazard |
| :--- | :--- | :--- |
| `async def route()` | Directly on the single **event loop thread**. | Any blocking I/O (`requests.get`, `time.sleep`, `psycopg2`, sync disk reads) or heavy CPU loop stalls **every** concurrent request in that worker. |
| `def route()` | Dispatched automatically by FastAPI/Starlette to an AnyIO worker **thread pool** (default 40 threads). | Safe for legacy sync SDKs, but caps concurrency at thread-pool size and adds thread context-switching overhead + GIL contention. |

---

## 2. Why Selected for SentinelRAG
- **I/O-Bound Gateway Workload:** SentinelRAG waits on Redis TCP sockets, PostgreSQL `asyncpg` binary sockets, and external LLM HTTP endpoints (`4.0s` timeout). Single-threaded async I/O lets each worker multiplex thousands of open waiting sockets cheaply.
- **Native AI Ecosystem:** `litellm`, `tiktoken`, `pgvector`, and `pydantic` are first-class in Python. Moving to Go/Rust would force CGO/FFI bindings or fragile custom provider adapters (see [09-design-decisions-and-tradeoffs.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md) for when Go/Rust wins).
- **Multi-Core Scaling:** Python's GIL only serializes bytecode execution inside a *single* process; production runs multiple Uvicorn worker processes across CPU cores (`--workers`), each with an isolated event loop and GIL.

---

## 3. Project Implementation & Configuration Details

### A. Lifespan Management (`app/main.py`)
In async Python, you cannot `await` connection pool pings at module import time because no event loop is running yet, and `@app.on_event("startup")` is deprecated. SentinelRAG uses `@asynccontextmanager async def lifespan(app)`:
- **Before `yield` (Startup):** Initializes `structlog` (`setup_logging()`), creates lazy SQLAlchemy (`db_manager.initialize()`) and Redis (`redis_manager.initialize()`) pools, and executes pre-flight `await db_manager.ping()` + `await redis_manager.ping()` so misconfigured URLs surface immediately in startup logs.
- **After `yield` (Shutdown):** Cleanly calls `await redis_manager.close()` (`aclose()` + `pool.disconnect()`) and `await db_manager.close()` (`engine.dispose()`).

### B. Dependency Injection DAG & `yield` Cleanup (`Depends`)
FastAPI resolves `Depends()` as a per-request Directed Acyclic Graph (with `use_cache=True` deduplication so shared sub-dependencies run once per request):
```python
async def get_async_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
        except Exception as exc:
            logger.error("database_session_transaction_failed", error=str(exc))
            await session.rollback()
            raise
        finally:
            await session.close()
```
- **Critical Detail:** Notice `app/db/session.py` in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md#L405-L420) does **not** auto-commit after `yield`—it only rolls back on exceptions and always closes. Write routes like `batch_ingest_documents` (`app/api/v1/documents.py`) explicitly call `await session.commit()`.

### C. Middleware Execution Order & `BaseHTTPMiddleware` Trade-Off
- In `app/main.py`, Starlette wraps middlewares from inside out (last registered executes **first**):
  `correlation_id_and_timing_middleware` $\to$ `CORSMiddleware` $\to$ `TokenBucketRateLimitMiddleware` $\to$ Route.
- **Timing & `contextvars` Hygiene:** `correlation_id_and_timing_middleware` uses `time.perf_counter()` (monotonic clock immune to NTP backwards jumps, unlike `time.time()`), calls `structlog.contextvars.clear_contextvars()` before `bind_contextvars(request_id=..., route=..., method=...)` so reused async task contexts never leak prior `request_id`s, and records latency even on unhandled 500 exceptions.
- **Starlette `BaseHTTPMiddleware` Caveat:** `TokenBucketRateLimitMiddleware` subclasses `BaseHTTPMiddleware`, which allocates internal anyio streams/response wrappers (~0.3–0.5ms overhead and historically fragile with streaming cancellation). At $50\text{k}+\text{ QPS}$ or active SSE streaming, convert it to a pure ASGI callable `async def __call__(self, scope, receive, send)`.

### D. Pydantic v2 (`pydantic-core`, `BaseSettings`, Validation)
- **Why 5–50x faster than v1:** Validation and `model_dump_json()` execute in compiled Rust (`pydantic-core`) using `jiter` (zero-copy JSON parser) instead of pure-Python recursion.
- **`app/core/config.py`:** Uses `Settings(BaseSettings)` with `case_sensitive=True`, `extra="ignore"`, numeric bounds (`ge`/`le`), and `@field_validator` (`@classmethod`) to fail fast at import time if `DATABASE_URL` lacks `postgresql+asyncpg://` or `REDIS_URL` lacks `redis://`/`rediss://`.
- **Edge Validation (`app/schemas/search.py`, `document.py`, `cache.py`):** Enforces `min_length=1536, max_length=1536` on `query_vector`/`embedding` so malformed vectors fail with `HTTP 422` in Rust before acquiring Postgres/Redis connections.
- **Best Practice Pitfalls:**
  - Always prefer `model.model_dump_json()` over `json.dumps(model.model_dump())` (avoids double traversal and `UUID`/`datetime` `TypeError`).
  - Use `@field_validator(mode="before")` for raw normalization and `mode="after"` (default) for typed checks; use `@model_validator(mode="after")` for cross-field invariants (e.g., validating `query_vector is not None` when `use_retrieval=True`—something SentinelRAG's `ChatCompletionRequest` could enforce to prevent `[0.0]*1536` fallbacks!).

---

## 4. Practical Problems & Solutions
1. **Event Loop Starvation in Production:**
   - *Diagnose:* Add a periodic `loop.call_later(0.01, ...)` probe measuring `actual - scheduled` lag (`asyncio_event_loop_lag_seconds`), or enable `PYTHONASYNCIODEBUG=1` in staging to flag callbacks $>100\text{ms}$.
   - *Fix:* Replace sync HTTP/DB drivers with async ones (`httpx`, `asyncpg`), offload unavoidable sync CPU/blocking work (e.g., giant `tiktoken` batches or pure-Python cosine loops over $>5\text{k}$ vectors) via `asyncio.to_thread()` or process pools.
2. **Stale `request_id` in Logs:**
   - *Cause:* Omitting `structlog.contextvars.clear_contextvars()` at the start of `correlation_id_and_timing_middleware`.

---

## 5. Quick Interview Questions
1. *Why `uvloop` over default `asyncio`, and when does `async def` freeze the server?*
2. *Explain how Starlette middleware registration order affects `429` error correlation IDs in `app/main.py`.*
3. *Why does `Depends` with `yield` protect database pools better than manual `try/finally` inside every endpoint?*
4. *Why does `app/core/config.py` validate `postgresql+asyncpg://` at startup instead of letting SQLAlchemy default?*

