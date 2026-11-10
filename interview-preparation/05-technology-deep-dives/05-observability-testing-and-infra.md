# Technology Deep Dive 5: Observability (`structlog`, Prometheus), `pytest-asyncio` & Production Infra (Docker, Nginx, CI/CD)

## 1. Fundamental Concepts & Internals

### Structured Logging (`structlog`) vs. Standard `logging`
- Standard `logging` outputs plaintext strings requiring brittle regex parsing and acquires `threading.Lock` on handlers (risking lock contention/blocking if `FileHandler` writes synchronously to disk).
- `structlog` uses a pure processor pipeline (`merge_contextvars` $\to$ `add_log_level` $\to$ `TimeStamper(fmt="iso")` $\to$ `JSONRenderer`) writing single-line JSON to `stdout` (so Docker/containerd handles non-blocking log shipping).
- `contextvars` provides task-local storage across `await` suspension points, preventing interleaved async tasks from corrupting `request_id`.

### Prometheus Pull Telemetry & Cardinality
- **4 Metric Types:** **Counter** (monotonic, resets on restart), **Gauge** (up/down snapshot), **Histogram** (cumulative `le` bucket counts + `_sum` + `_count` enabling cross-replica `histogram_quantile()` via linear interpolation), **Summary** (client-side quantiles, cannot aggregate across pods).
- **Pull vs. Push:** Prometheus scrapes `GET /metrics` every ~15–30s. Pull decouples app runtime from collector outages, provides free liveness detection (`UP == 0`), and prevents traffic spikes from DDoS-ing the monitoring server with push floods.
- **Cardinality Explosion:** Every unique `(metric_name, label_values)` tuple creates an active time series in TSDB RAM. Putting unbounded values (`request_id`, `user_id`, IP, raw query) in Prometheus labels OOM-crashes TSDB. Keep labels strictly bounded (`endpoint`, `status`, `result`, `model`, low-cardinality `tenant`).

### `pytest-asyncio` & `httpx.ASGITransport`
- `ASGITransport(app=app)` invokes the FastAPI ASGI callable directly in memory—zero TCP ports opened, sub-millisecond dispatch, parallel-safe in CI (`pytest -n auto`).
- `app.dependency_overrides[dep] = mock_fn` cleanly injects `FakeRedis` / `AsyncMock` without brittle global monkeypatching—always clear in `finally: app.dependency_overrides.clear()`.

### Container Security, Linux PID 1 & Nginx Socket Tuning
- **Linux Containers:** Isolated via **Namespaces** (`pid`, `net`, `mnt`, `ipc`, `uts`, `user` — visibility) and **cgroups v2** (CPU/RAM/`pids.max` caps — resource limits), sharing the host kernel. Running as `UID 0` (`root`) means a container escape gives host root.
- **PID 1 Problem:** Inside Linux containers, PID 1 must reap orphaned zombie processes (`wait()`) and explicitly trap `SIGTERM` (kernel suppresses default termination signal actions for PID 1). Shell wrappers (`CMD ./run.sh`) swallow `SIGTERM`, causing 10s hangs followed by ungraceful `SIGKILL`. Using exec-form `CMD ["uvicorn", ...]` runs Uvicorn directly as PID 1 for graceful connection draining.
- **Nginx `epoll` & Streaming:** Event-driven workers handle thousands of keepalive sockets without Apache's thread-per-connection memory bloat. Turning `proxy_buffering off` and `proxy_cache off` prevents Nginx from holding SSE token chunks in 4K/8K buffers until stream end.

---

## 2. Project Implementation & Configuration Details

### A. Logging & Prometheus Telemetry (`app/core/logging.py`, `app/core/telemetry.py`)
- **Logging:** Uses `ConsoleRenderer` only when `ENVIRONMENT == "development" and DEBUG` is True; otherwise renders structured JSON. Silences noisy `uvicorn.access`, `sqlalchemy.engine`, and `asyncio` loggers at `WARNING` level.
- **PII Caution:** Notice `chat.py` line 4555 logs `query=query_text` in `chat_completion_served_from_cache`—in strict GDPR/PCI-DSS deployments, redact or hash raw `query_text` in logs so PII isn't persisted in log aggregators!
- **4 Verified Prometheus Metrics:**
  1. `http_request_duration_seconds` (`Histogram`, labels `["endpoint", "status"]`, custom buckets `(0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 10.0)` calibrated from 5ms cache hits up to 10s LLM calls).
  2. `cache_operations_total` (`Counter`, labels `["result", "tenant"]`).
  3. `tokens_consumed_total` (`Counter`, labels `["tenant", "model"]`).
  4. `cost_saved_usd_total` (`Counter`, labels `["tenant"]`).
- **Multi-Worker Note:** With `--workers > 1`, `prometheus_client` needs `PROMETHEUS_MULTIPROC_DIR` (shared `tmpfs` `mmap`) so scrapes don't see only one random worker's counters! Also note `tenant` labels must be bounded to known tenant IDs (or tiered) to avoid cardinality explosion at massive B2B scale.

### B. Automated Test Suite (`tests/` — 12 modules, 44 tests)
- `asyncio_mode = "auto"` in `pyproject.toml`.
- Covers `/healthz` (`200` vs `503`), Token Bucket Lua depletion/refill/`429` headers (`FakeRateLimitRedis`), IEEE 754 roundtrip & cosine distance boundaries (`FakeRedis`), batch document `422` vector dimension checks, RRF math & tie-breaking, hybrid search mocking, `CircuitBreaker` 5-failure trip + `HALF_OPEN` recovery, Tenacity transient filtering, router primary/fallback cascades, `tiktoken` math, and E2E `/api/v1/chat/completions` + `/metrics`.
- **Important Caveat:** All 44 tests mock Postgres/Redis/LiteLLM (`AsyncMock`, `FakeRedis`, `patch("litellm.acompletion")`)—they validate logic, contracts, and in-memory execution paths, not real live DB query plans or live provider latencies.

### C. Multi-Stage `Dockerfile` (`uv` + UID 10001) & `docker-compose.prod.yml`
- **Stage 1 (`builder`, `python:3.11-slim`):** Copies `uv` from `ghcr.io/astral-sh/uv:latest`, installs `build-essential` & `libpq-dev` for C-extensions (`asyncpg`, `hiredis`), runs `uv sync --frozen --no-dev --no-install-project` with BuildKit cache mount `--mount=type=cache,target=/root/.cache/uv` and `UV_COMPILE_BYTECODE=1`.
- **Stage 2 (`runner`, `python:3.11-slim`):** Installs *only* `libpq5` + `curl` (no compilers!), creates non-root `appuser:appgroup` (`UID/GID 10001`, `-s /bin/false`, `-M`), copies `.venv` and `app/` with `--chown=appuser:appgroup`, switches `USER appuser`, sets `HEALTHCHECK` against `/healthz`, and launches exec-form `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--loop", "uvloop", "--no-access-log"]`.
- **Network Isolation (`docker-compose.prod.yml`):** Only `nginx` publishes ports (`80`, `443`); `api`, `db` (`pgvector/pgvector:pg16`), and `cache` (`redis:7.2-alpine` with `--appendonly yes --maxmemory 512mb --maxmemory-policy volatile-lru`) sit strictly on internal bridge `sentinel_prod_network` with zero exposed host ports.

### D. Nginx (`nginx/nginx.conf`, `nginx/conf.d/sentinel.conf`) & Host/CI Hardening
- **`nginx.conf`:** `worker_processes auto`, `worker_connections 2048`, `use epoll`, `sendfile on`, `tcp_nopush on`, `tcp_nodelay on` (disables Nagle's 200ms buffering on keepalive sockets), `server_tokens off`, `client_max_body_size 50M`, JSON access log including `$http_x_request_id` and `$upstream_response_time`.
- **`sentinel.conf`:** HTTP $\to$ HTTPS `301` (preserving Certbot ACME path), TLS 1.2/1.3 AEAD ciphers, `ssl_session_tickets off` (protects **Perfect Forward Secrecy** if a ticket key leaks), OCSP stapling (`ssl_stapling on`), HSTS preload header, `keepalive 64`, and `proxy_buffering off` + `proxy_cache off`.
  - *Security Improvement Opportunity:* `sentinel.conf` currently exposes `location = /metrics` publicly through Nginx; in production, restrict `/metrics` via internal IP allowlist (`allow 10.0.0.0/8; deny all;`) or bearer auth so attackers cannot enumerate tenant traffic/models.
- **CI/CD (`.github/workflows/deploy.yml`):**
  1. **Job 1 (`test`):** Spins up ephemeral `redis:7.2-alpine` & `pgvector/pgvector:pg16` health-checked service containers, runs `ruff check`, `ruff format --check`, `alembic upgrade head`, and `pytest`.
  2. **Job 2 (`build-and-push`):** QEMU + Buildx multi-arch (`linux/amd64`, `linux/arm64`) push tagged by short git SHA + `latest`.
  3. **Job 3 (`deploy`):** SSH to host, pulls SHA image, runs **pre-deploy one-off migration container** (`docker run --rm ... alembic upgrade head`), runs `docker compose up -d --remove-orphans`, polls `/healthz` up to 60s, and prunes dangling images.
- **`scripts/init_server.sh`:** Configures Ubuntu `ufw` (default deny, only rate-limited SSH 22, HTTP 80, HTTPS 443), `fail2ban`, and Docker daemon `"no-new-privileges": true` + `"live-restore": true` + log rotation (`50m × 5`).

---

## 3. Quick Interview Questions
1. *Why are SentinelRAG's Prometheus histogram buckets `(0.005, ..., 10.0)` instead of Prometheus defaults, and how does `histogram_quantile` break if buckets are wrong?*
2. *Why run `alembic upgrade head` in a one-off CI/CD deployment container rather than inside the Dockerfile entrypoint?*
3. *Why does `sentinel.conf` set `ssl_session_tickets off` and `proxy_buffering off`?*
4. *How does `scripts/init_server.sh` + `Dockerfile` enforce least-privilege defense-in-depth?*

