# 10. Follow-Up & Cross-Questioning Chains

Interviewers rarely stop at your first answer—they pull on each technical thread until they hit the boundary of your real understanding. Practice answering these 6 progressive chains end-to-end.

---

## Chain 1: Semantic Caching & Redis Internals

- **Primary Q:** *Why did you use Redis for semantic caching instead of an in-memory Python LRU or calling the LLM every time?*
  - **A:** In-memory caches fragment across multi-worker processes (`1/N` hit rate) and duplicate gigabytes of vector RAM across containers. Centralized Redis shares cached generations across all workers in ~1ms over the Docker bridge while avoiding multi-second LLM latency and token cost.
- **Follow-up 1:** *If two questions are phrased slightly differently, how does Redis match them?*
  - **A:** `RedisSemanticCache` runs two stages (`app/services/cache.py`): Stage 1 checks `SHA-256(query.strip().lower())` via `HGETALL` in $O(1)$. On miss, Stage 2 reads `SMEMBERS cache:{tenant_id}:keys`, pipelines `HGETALL` across tenant entries, unpacks stored embeddings, and finds the highest cosine similarity—returning a hit if `cosine_distance = 1 - similarity <= 0.08`.
- **Follow-up 2:** *Why `0.08` specifically? What happens at `0.20` or `0.01`?*
  - **A:** Distance `0.08` (similarity `0.92`) balances paraphrase recall against semantic safety. At `0.20`, queries with high lexical overlap but different entities—like *"What is Plan A's deductible?"* vs. *"What is Plan B's deductible?"*—collide and return false answers. At `0.01`, valid paraphrases miss and waste tokens.
- **Follow-up 3:** *How are those 1536-dimensional embeddings stored in Redis? Why not JSON?*
  - **A:** JSON text for 1536 floats takes ~28–35 KB and expensive string parsing. `serialize_embedding()` uses `struct.pack` to encode IEEE 754 32-bit floats into `6,144` bytes, then Base64-encodes to `8,192` ASCII characters because `RedisManager` sets `decode_responses=True` (which otherwise fails UTF-8 decoding on raw binary floats). *(Bonus point: using explicit `<` little-endian `f"<{len}f"` protects mixed ARM64/x86 fleets).*
- **Follow-up 4:** *Each cache entry expires after 24 hours (`86400s`), so how do you stop `cache:{tenant_id}:keys` from filling with dead keys forever?*
  - **A:** Redis TTLs delete the individual hashes, not members inside a separate Set. During Stage 2's pipelined `HGETALL`, any missing (`{}`) or corrupt hash is collected into `expired_keys` and pruned lazily via `SREM`, plus the index set itself carries `ttl * 2`.
- **Follow-up 5:** *What happens if 10,000 entries accumulate for one tenant, or if 200 concurrent users miss on the same query simultaneously?*
  - **A:** Past ~5,000 active entries per tenant, Python's $O(N \cdot D)$ scan slows down—so we migrate Tier 2 to **RediSearch `FT.SEARCH` HNSW** (`KNN`). For concurrent cache misses (cache stampede), we add a Redis `SET lock NX EX` singleflight lock so only one worker invokes the LLM while waiters read the populated entry.
- **Follow-up 6:** *What happens if Redis crashes completely?*
  - **A:** `get_or_lookup` and `set` catch connection errors and degrade transparently to `CACHE_MISS` / pass-through, keeping the gateway serving via Postgres + LLM.

---

## Chain 2: PostgreSQL Single-Store & Hybrid Search

- **Primary Q:** *Why use PostgreSQL + `pgvector` instead of Pinecone or Qdrant?*
  - **A:** Keeping relational metadata, JSONB, 1536d embeddings, and `TSVECTOR` in a single table (`document_chunks`) gives ACID transactions on insert/delete and eliminates the dual-write drift and operational sprawl of syncing Postgres with an external vector SaaS.
- **Follow-up 1:** *Why HNSW instead of IVFFlat in `pgvector`?*
  - **A:** IVFFlat clusters via k-means (`lists`), requiring representative training data upfront (fails on empty tables) and degrading as new data shifts centroids. HNSW builds a multi-layer proximity skip-list graph offering $O(\log N)$ lookup, $>95\%$ recall, and zero-retraining incremental inserts.
- **Follow-up 2:** *What do `m=16`, `ef_construction=64`, and `vector_cosine_ops` actually control?*
  - **A:** `m=16` bounds bidirectional links per graph node (recall vs. RAM/build size); `ef_construction=64` bounds the dynamic candidate queue during index creation (graph quality vs. build speed); runtime `hnsw.ef_search` tunes query-time recall vs. latency. Crucially, `vector_cosine_ops` binds the index to `<=>`—querying with `<->` (L2) forces Postgres into a full `Seq Scan`.
- **Follow-up 3:** *Why did you execute dense and sparse queries sequentially in `SearchRepository` instead of `asyncio.gather()`?*
  - **A:** An `AsyncSession` uses one half-duplex `asyncpg` TCP socket. Calling `session.execute()` concurrently from two coroutines triggers `asyncpg.InterfaceError: cannot perform operation: another operation is in progress`. Running them sequentially avoids checking out two pool connections per request while still completing in milliseconds.
- **Follow-up 4:** *Why run two queries + RRF instead of one SQL query with `WHERE search_vector @@ ... ORDER BY embedding <=> ... LIMIT 10`?*
  - **A:** Postgres cannot traverse both a GIN index and an HNSW index in a single physical scan. If it picks HNSW and filters rare lexical matches afterward, it traverses huge swaths of the graph slowly; if it picks GIN, it brute-forces vector distances across all matches. Two independent index scans merged in Python via RRF ($k=60$) keep both index scans fast.
- **Follow-up 5:** *Why RRF (`k=60`) instead of `0.7 * dense + 0.3 * sparse`?*
  - **A:** Cosine distance ($[0, 2]$) and `ts_rank_cd` ($[0, \infty)$) are incompatible scales. Min-max normalization breaks on outliers or weak lexical queries. RRF operates purely on 1-based ordinal ranks, and `k=60` dampens single-list #1 dominance (~1.6% gap between rank 1 and 2) so consensus hits across both modalities win.
- **Follow-up 6:** *What happens if you add PgBouncer in front of PostgreSQL as pods scale?*
  - **A:** You run PgBouncer in **transaction pooling** mode so thousands of app sockets multiplex onto ~`2 × CPU cores` Postgres backends—*and* you must set `statement_cache_size=0` on `asyncpg` because server connections switch clients across transactions, invalidating connection-local prepared statements.

---

## Chain 3: Distributed Rate Limiting & Lua Atomicity

- **Primary Q:** *Why implement rate limiting in a Redis Lua script instead of Python middleware or standard Redis `INCR`?*
  - **A:** Python in-memory limiters break behind multi-worker Uvicorn/load balancers. `INCR + EXPIRE` fixed windows allow 2x boundary bursts. `GET` then `SET` creates check-then-act race conditions, and `MULTI/EXEC` cannot run conditional `if tokens >= requested` math on values read inside the same transaction. Lua runs atomically in Redis's single thread in one RTT.
- **Follow-up 1:** *Why call `redis.call('TIME')` inside Lua instead of passing `time.time()` from FastAPI?*
  - **A:** Container clocks drift slightly even with NTP. Using Redis's single clock guarantees monotonic, consistent elapsed `delta` calculations across all API workers—plus `math.max(0, now - last_updated)` protects against tiny backward clock steps.
- **Follow-up 2:** *What if Redis clears its script cache (`SCRIPT FLUSH` or restart)?*
  - **A:** `TokenBucketLimiter` caches the `SCRIPT LOAD` SHA-1 and invokes `EVALSHA` to avoid resending script text; if Redis replies `NoScriptError`, it clears `_script_sha` and falls back cleanly to `EVAL` without dropping the request.
- **Follow-up 3:** *What if Redis itself dies—fail open or fail closed?*
  - **A:** `rate_limiter.py` fails **open** (logs critical error and allows traffic) because a cache glitch shouldn't take down the entire API—relying on Nginx edge limits (`limit_req_zone`) as a secondary shield. (Note: for strict financial billing APIs where overspend > availability, you'd configure fail-closed instead).
- **Follow-up 4:** *How would you extend this from request-count limiting to token-aware (TPM) limiting?*
  - **A:** Pass `requested = count_messages_tokens(...)` (pre-flight) or deduct actual `total_tokens` post-completion via Lua so massive 8k-token prompts consume proportional bucket capacity against smaller 20-token prompts.

---

## Chain 4: Resilient LLM Routing & Circuit Breaker

- **Primary Q:** *Why build a Circuit Breaker when `try/except` + retries can already fall back to OpenAI?*
  - **A:** Without a Circuit Breaker, during a 15-minute Groq outage every single request wastes `3 × 4s` timeouts + exponential sleep before failing over—saturating worker coroutines and spiking latency across the fleet. Once 5 consecutive failures trip the breaker to `OPEN`, subsequent calls bypass Groq in `0ms` for 60s.
- **Follow-up 1:** *Why do you need `HALF_OPEN` instead of flipping straight from `OPEN` to `CLOSED` after 60 seconds?*
  - **A:** Jumping straight to `CLOSED` slams a just-recovering provider with 100% of queued production traffic simultaneously, tripping it right back down. `HALF_OPEN` sends a canary probe first and only restores `CLOSED` if it succeeds.
- **Follow-up 2:** *What happens if 50 requests arrive concurrently the millisecond the 60s cooldown expires?*
  - **A:** `asyncio.Lock` serializes state transitions cleanly (`threading.Lock` would freeze the event loop!). *Honest gap check:* in current `circuit_breaker.py`, once state flips to `HALF_OPEN`, subsequent callers arriving while the canary request is still awaiting network I/O also see `HALF_OPEN` unless we add a `_probe_in_flight` boolean flag to limit probes strictly to one concurrent request.
- **Follow-up 3:** *Why Full Jitter in Tenacity (`wait_random_exponential`), and which HTTP codes do you NOT retry?*
  - **A:** Full Jitter `Uniform(0, min(cap, base * 2^attempt))` eliminates synchronized retry waves better than Equal or no jitter. `is_transient_error()` retries `429, 502, 503, 504`, timeouts, and connection errors, while failing immediately on permanent `400, 401, 403, 404` client errors.
- **Follow-up 4:** *What happens if you enable SSE streaming (`stream=True`) and Groq drops mid-stream after sending 50 tokens?*
  - **A:** Once HTTP `200` headers and initial SSE chunks have flushed to the client, you can't rewrite the HTTP status or silently switch providers mid-sentence. You must emit a structured in-band SSE error event, close the stream cleanly, and handle `asyncio.CancelledError` on client disconnect so upstream token billing stops immediately.

---

## Chain 5: FastAPI, Concurrency & SQLAlchemy Internals

- **Primary Q:** *How does one Uvicorn process handle thousands of concurrent users in Python?*
  - **A:** `uvloop` binds `libuv` (`epoll` / `kqueue`) to run a single-threaded cooperative event loop; `async def` routes suspend at non-blocking `await` I/O (`asyncpg`, `redis.asyncio`, `httpx`), multiplexing thousands of open sockets with minimal RAM.
- **Follow-up 1:** *What breaks if someone adds `requests.get()` or a giant pure-Python loop inside `chat_completions()`?*
  - **A:** The single event loop thread freezes, stalling all concurrent requests on that worker until the blocking call completes. Sync I/O should run in `asyncio.to_thread()` or `def` routes, and heavy CPU math in `ProcessPoolExecutor` / compiled C/Rust (`numpy`, `tiktoken`).
- **Follow-up 2:** *Why did `app/db/session.py` set `expire_on_commit=False` and `pool_pre_ping=True`?*
  - **A:** `expire_on_commit=False` stops SQLAlchemy from expiring ORM fields on `commit()`, avoiding synchronous lazy-load attempts that crash with `MissingGreenlet` in async code. `pool_pre_ping=True` runs `SELECT 1` on checkout to discard firewall-dropped half-open TCP sockets transparently.
- **Follow-up 3:** *What happens if an endpoint raises an exception after inserting half a batch?*
  - **A:** `get_async_session()` catches the exception after `yield`, runs `await session.rollback()`, re-raises, and executes `await session.close()` in `finally`—never leaking dirty connections back to the pool.

---

## Chain 6: Deployment, Nginx, Observability & Testing

- **Primary Q:** *Walk me through how SentinelRAG ships safely to production and stays observable.*
  - **A:** GitHub Actions Job 1 runs `ruff`, `alembic upgrade head`, and 44 async `pytest` cases against ephemeral `pgvector` + `redis` service containers; Job 2 builds multi-arch `amd64/arm64` images via Docker Buildx (`uv` multi-stage build running as non-root `UID 10001`); Job 3 runs pre-deploy migrations in a one-off container, updates Compose with `--remove-orphans`, and polls `/healthz`.
- **Follow-up 1:** *Why run `alembic upgrade head` in a one-off container instead of `CMD` inside the Dockerfile?*
  - **A:** If 4 API containers boot simultaneously running migrations in `CMD`, they race on PostgreSQL DDL locks—and runtime containers shouldn't hold DDL `ALTER TABLE` database permissions anyway.
- **Follow-up 2:** *Why does `Dockerfile` use exec-form `CMD ["uvicorn", ...]` and `USER appuser` (`10001`)?*
  - **A:** Exec form puts Uvicorn directly at **PID 1** so `SIGTERM` from `docker stop` drains active requests cleanly instead of being swallowed by a `/bin/sh` wrapper. `UID 10001` drops root capabilities and removes compilers from the runner stage so container breakouts can't grant host root.
- **Follow-up 3:** *Why do all 44 tests pass in ~20 seconds without real API keys, and what is the risk?*
  - **A:** `ASGITransport(app=app)` dispatches ASGI requests in memory without TCP overhead while `app.dependency_overrides` + `AsyncMock` / `FakeRedis` simulate Postgres, Redis, and LiteLLM deterministically. The trade-off is **mock drift**—mitigated by validating schemas with Pydantic v2 and adding CI integration tests against real `pgvector` + `redis` containers.

---
**Next:** [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md)

