# Implementation Deep Dive 1: App Lifecycle, Config, Connections, Middleware & Error Handling

**Covered Modules (Verified in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md)):**
`app/core/config.py` | `app/core/logging.py` | `app/core/database.py` | `app/db/session.py` | `app/core/redis.py` | `app/core/exceptions.py` | `app/core/telemetry.py` | `app/main.py`

---

## 1. Type-Safe Configuration (`app/core/config.py`)

`Settings(BaseSettings)` validates environment variables once at import time (`settings = Settings()`), preventing delayed `NoneType` runtime failures under traffic.

- **Key Validators:**
  ```python
  @field_validator("DATABASE_URL")
  @classmethod
  def validate_asyncpg_protocol(cls, v: str) -> str:
      if not v.startswith("postgresql+asyncpg://"):
          raise ValueError(...)
      return v
  ```
  *Why it matters:* Without `+asyncpg://`, SQLAlchemy defaults to a synchronous driver (`psycopg2`) and silently blocks the single-threaded ASGI event loop on every SQL query.
- **Verified Defaults You Must Know:**
  - `DB_POOL_SIZE = 20`, `DB_MAX_OVERFLOW = 10`, `DB_TIMEOUT_SECONDS = 30`
  - `REDIS_POOL_SIZE = 50`, `REDIS_TIMEOUT_MS = 500`
  - `SEMANTIC_CACHE_THRESHOLD = 0.08`, `SEMANTIC_CACHE_TTL_SECONDS = 86400`
  - `RATE_LIMIT_DEFAULT_CAPACITY = 100`, `RATE_LIMIT_DEFAULT_REFILL_RATE_PER_SEC = 10`
  - `DEFAULT_PRIMARY_MODEL = "groq/llama-3.3-70b-versatile"`, `FALLBACK_1 = "openai/gpt-4o-mini"`, `FALLBACK_2 = "mistral/mistral-small-latest"`
  - `CIRCUIT_BREAKER_FAILURE_THRESHOLD = 5`, `RECOVERY_TIME_SECONDS = 60`, `TIMEOUT_SECONDS = 4.0`

---

## 2. Database & Redis Connection Management (`database.py`, `session.py`, `redis.py`)

### Async SQLAlchemy Engine (`app/db/session.py` & `app/core/database.py`)
- `create_async_engine(...)` is **synchronous**—it allocates the pool manager without opening TCP connections until `ping()` or query checkout.
- `pool_pre_ping=True` issues `SELECT 1` on checkout to detect silently dropped idle TCP connections (e.g., cloud NAT timeouts) before user queries hit `ConnectionResetError`.
- `pool_recycle=300` refreshes connections every 5 minutes.
- `expire_on_commit=False` prevents SQLAlchemy from marking attributes stale after `commit()`, avoiding synchronous lazy-load attempts (`MissingGreenlet`).
- `get_async_session()` yields an `AsyncSession` wrapped in `try / except (rollback + raise) / finally (close)` so exceptions never return dirty uncommitted transactions to the pool.
- *Module Note:* `database.py` imports `engine` and `async_session_factory` from `app/db/session.py`, and exposes `db_manager.ping()` returning `(bool, latency_ms)`.

### Async Redis Pool (`app/core/redis.py`)
- `RedisManager.initialize()` builds an `aioredis.ConnectionPool.from_url(..., max_connections=50, decode_responses=True, socket_timeout=0.5, socket_connect_timeout=0.5)`.
- Because `decode_responses=True`, Redis reads automatically decode UTF-8 `str` instead of `bytes`—dictating why binary `struct.pack` floats in `cache.py` are Base64-encoded.
- `close()` uses `await self.client.aclose()` followed by `await self.pool.disconnect()`.

---

## 3. ASGI Entrypoint, Lifespan & Middleware Stack (`app/main.py`)

### Lifespan Startup & Shutdown
```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    setup_logging()
    db_manager.initialize()
    redis_manager.initialize()
    db_healthy, db_lat = await db_manager.ping()
    redis_healthy, redis_lat = await redis_manager.ping()
    ...
    yield
    await redis_manager.close()
    await db_manager.close()
```
- **Why `ping()` during startup?** Catches misconfigured DB/Redis credentials or unreachable containers immediately in startup telemetry rather than waiting for the first customer request. (Note: it logs readiness rather than aborting startup, letting `/healthz` return `503` if a subsystem is temporarily booting).

### Middleware Order & Correlation Context (`correlation_id_and_timing_middleware`)
Remember Starlette wraps `add_middleware` from inside out, so the actual inbound execution flow in `app/main.py` is:
1. `correlation_id_and_timing_middleware`:
   - Extracts `X-Request-Id` or generates `f"req_{uuid.uuid4().hex[:16]}"`, attaches to `request.state.request_id`.
   - Calls `structlog.contextvars.clear_contextvars()` then `bind_contextvars(request_id=..., route=..., method=...)`.
   - Measures latency with `time.perf_counter()`, observes `HTTP_REQUEST_DURATION_SECONDS`, and sets `X-Request-Id` and `X-Sentinel-Latency-Ms` response headers.
2. `CORSMiddleware`: Exposes custom gateway headers (`X-Sentinel-*`, `X-Request-Id`, `X-RateLimit-*`, `Retry-After`).
3. `TokenBucketRateLimitMiddleware`: Uses `redis_getter=lambda: redis_manager.client` (lazy lambda because `redis_manager.client` is still `None` at import time when `app.add_middleware` runs, before `lifespan` executes!).

### Diagnostics Endpoints
- **`GET /healthz`:** Calls `await db_manager.ping()` and `await redis_manager.ping()`. Returns `200 OK` (`status="healthy"`) when both succeed or `503 Service Unavailable` (`status="degraded"`) when either fails—distinguishing upstream dependency failure (`503`) from application bugs (`500`) so Kubernetes/load balancers unroute traffic instead of paging for app crashes.
- **`GET /metrics`:** Returns `metrics_response()` (`generate_latest()` with `CONTENT_TYPE_LATEST`).
- **`docs_url` / `redoc_url`:** Automatically disabled (`None`) when `settings.ENVIRONMENT == "production"`.

---

## 4. Structured Error Handling (`app/core/exceptions.py`)

- `ProblemDetailException` is the base class for `RateLimitExceededException` (`429`) and `DatabaseConnectionException` (`503`).
- `problem_detail_exception_handler` serializes RFC 7807 JSON (`type`, `title`, `status`, `detail`, `instance`, `request_id`, `invalid_params`) with `Content-Type: application/problem+json` and logs a `structlog` warning.
- **Implementation Nuance:** `TokenBucketRateLimitMiddleware` returns its own RFC 7807 `JSONResponse` directly instead of raising `RateLimitExceededException` because app-level `add_exception_handler` callbacks sit inside `ServerErrorMiddleware`/`ExceptionMiddleware` rather than wrapping custom outer middlewares cleanly.

---
**Next:** [02-rate-limiter-and-semantic-cache.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md)

