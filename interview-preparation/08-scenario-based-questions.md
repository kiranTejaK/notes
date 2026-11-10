# 08. Scenario-Based Interview Questions

Realistic operational, failure, and production-debugging scenarios tailored specifically to SentinelRAG's actual code paths (`app/main.py`, `rate_limiter.py`, `cache.py`, `search.py`, `circuit_breaker.py`, `router.py`, `chat.py`).

---

## Scenario 1: Redis Completely Crashes or Becomes Unreachable

**Interviewer:** *"Suppose your `sentinel_cache` Redis container crashes or suffers a network partition during peak traffic. Does your entire AI gateway go down?"*

### Layer 1 – Short Answer
> "No—SentinelRAG stays available in degraded mode. Both the Redis Token Bucket rate limiter and `RedisSemanticCache` are explicitly designed to **fail open**, while `TokenAnalyticsTracker` falls back to in-memory accumulators. Meanwhile, `/healthz` flips to HTTP 503 to alert orchestration/monitoring."

### Layer 2 & 3 – Exact Code Propagation
1. **`GET /healthz` (`app/main.py`):** `await redis_manager.ping()` returns `(False, 0.0)`, flipping `/healthz` from `200 OK` to `503 Service Unavailable` (`redis.status = "down"`).
   - *Design Tension to Mention:* If Kubernetes points its **readiness probe** at `/healthz`, Kubernetes will pull pods out of rotation even though `chat/completions` can still serve traffic! A senior fix is splitting `/healthz/live` (process alive), `/healthz/ready` (Postgres required; Redis optional degraded flag), or relying on Prometheus alerts instead of pulling all pods when Redis blips.
2. **Rate Limiter (`app/core/rate_limiter.py`):** `TokenBucketLimiter.check()` catches the Redis timeout/connection error, logs `rate_limit_redis_evaluation_error`, and returns `RateLimitResult(allowed=True, ...)`—paired with Nginx coarse IP rate limits (`limit_req_zone`) as outer defense-in-depth.
3. **Semantic Cache (`app/services/cache.py`):** `get_or_lookup()` catches errors in both Tier 1 (`exact_lookup_error`) and Tier 2 (`semantic_cache_lookup_failed`) and returns `CACHE_MISS`. In `chat.py`, Stage 4 (`cache_service.set`) also catches write failures (`cache_write_failed_continuing`) so responses still return `200 OK`.
4. **Latency Impact:** Every request pays `socket_timeout` (`500ms` per Redis call!) until Redis recovers or you add a Redis-level circuit breaker—a great follow-up optimization to point out!

---

## Scenario 2: PostgreSQL Becomes Unavailable or Crawls Under Heavy Ingestion

**Interviewer:** *"What happens if PostgreSQL goes down—or if massive document ingestion slows `pgvector` HNSW queries from 15ms to 2,000ms?"*

### Layer 1 – Short Answer
> "If PostgreSQL is completely unreachable, `/healthz` returns 503, `/documents/batch` and `/search/hybrid` fail and roll back cleanly, while `/chat/completions` behaves differently by path: cache hits still return in under 20ms without touching Postgres, and cache misses with `use_retrieval=True` catch the search error and fall back to unaugmented LLM responses."

### Layer 2 – Exact Failure & Slowdown Mechanics
- **If Postgres is Down:**
  - `get_async_session()` catches DB connection errors, attempts rollback, and closes cleanly.
  - `/api/v1/documents/batch` and `/api/v1/search/hybrid` return `500` (could be mapped via `DatabaseConnectionException` $\to$ RFC 7807 `503`).
  - `chat_completions()` (`app/api/v1/chat.py`): Cache hits (Stage 1) never call `SearchRepository`, so cached queries still succeed `200 OK`. Cache misses with `use_retrieval=True` hit `except Exception as search_err:` (`hybrid_retrieval_degraded_continuing_unaugmented`) and call the LLM without RAG context.
- **If Postgres Slows Down (Connection Pool Saturation):**
  - Holding `AsyncSession` open for `2s` per query across 30 concurrent requests hits `pool_size=20 + max_overflow=10 = 30`, queuing subsequent checkouts until `pool_timeout=30s` raises `TimeoutError` (`QueuePool limit reached`).
  - **Note an existing advantage in `chat.py`:** Retrieval finishes quickly before Stage 3 `router_service.complete()` runs, *though* `session` stays open inside FastAPI's `Depends(get_async_session)` until the route returns! Closing/committing the read session explicitly *before* awaiting the 1–3s LLM call prevents slow LLM responses from pinning Postgres pool slots!
  - **Mitigations for HNSW Slowdown:** Enforce `SET LOCAL statement_timeout = '100ms'` inside `SearchRepository`, degrade to sparse-only GIN results on dense timeout, and route read queries to a PostgreSQL read replica.

---

## Scenario 3: Duplicate Concurrent Requests & Semantic Cache Stampede

**Interviewer:** *"500 users ask the exact same new question within 100 milliseconds. How does your system behave right now, and how would you prevent duplicate LLM work?"*

### Layer 1 – Short Answer
> "Right now, all 500 concurrent requests miss the cache before the first LLM response finishes, executing 500 parallel RAG + LLM calls (cache stampede), though Redis writes stay clean because `HSET` is idempotent. To prevent duplicate upstream calls, I would add a Redis singleflight lock (`SET cache:lock:{tenant}:{hash} NX EX 15`) or in-process `asyncio.Future` coalescing."

### Layer 2 – How to Implement Singleflight Deduplication
1. Compute `exact_key = cache:{tenant_id}:{sha256}`.
2. On miss, attempt `SET lock:{exact_key} 1 NX EX 15`:
   - **Winner (returns `True`):** Executes hybrid search + LLM call, writes result to Redis, releases lock (`DEL`).
   - **Waiters (return `None`):** Sleep/poll briefly (e.g., 50ms backoff or wait on Redis Pub/Sub channel for that key) and re-read `HGETALL exact_key` once populated—cutting 500 LLM calls down to **1**.
3. For popular expiring entries, add **TTL Jitter** (`86400 + random(-600, 600)`) to avoid synchronized cache avalanches, or **Probabilistic Early Refresh (`XFetch`)**.

---

## Scenario 4: Investigating a Sudden Slow API (`p99` Spike in Production)

**Interviewer:** *"Alerts fire that p95/p99 latency on `/api/v1/chat/completions` jumped from 400ms to 4,500ms. Walk me through your investigation."*

### Layer 1 – Short Answer
> "I'd correlate three signals: Prometheus `http_request_duration_seconds` + `cache_operations_total` to see whether cache hits dropped or upstream latency rose; Nginx JSON access logs (`$request_time` vs `$upstream_response_time`) to isolate edge vs. app; and `structlog` JSON logs filtered by `X-Request-Id` (`req_*`) to pinpoint the exact slow stage."

### Layer 2 – Diagnostic Runbook
1. **Check Nginx vs. Gateway (`nginx.conf`):** Compare `request_time` vs `upstream_response_time`. If `request_time >> upstream_response_time`, slow clients or network are the bottleneck; if equal, the delay is inside FastAPI/DB/Redis/LLM.
2. **Check Prometheus `/metrics`:**
   - Did `cache_operations_total{result="hit"}` plummet? (Unexpected tenant cache invalidation or `query_vector=None` client bug skipping Stage 2 semantic matches!)
   - Check `tokens_consumed_total` by `model`—did traffic shift to fallback `gpt-4o-mini` because Groq tripped the circuit breaker?
3. **Trace Specific Slow Requests via `X-Request-Id`:**
   - Grep JSON logs for a slow `execution_latency_ms` request ID:
     - `semantic_cache_exact_hit` vs `semantic_cache_lookup_failed` (`latency_ms` inside `get_or_lookup` reveals if a tenant's cache set grew $>5,000$ entries!).
     - `hybrid_retrieval_augmented_prompt` timestamp delta vs `llm_router_invocation` reveals Postgres query slowdown (verify with `pg_stat_activity` and `EXPLAIN ANALYZE` to confirm HNSW/GIN index usage vs `Seq Scan`).
     - `transient_error_detected_retrying` warnings reveal upstream provider retries (`3 attempts × 4s timeout + jitter ≈ 12s+`).
4. **Check Event Loop Saturation:** If even `/healthz` latency spiked, something is blocking the Python event loop (sync I/O or oversized Tier 2 cosine scan across huge tenant cache sets).

---

## Scenario 5: Sudden 10x Traffic Spike & Token-Cost Abuse

**Interviewer:** *"A single enterprise customer accidentally deploys an infinite loop sending 5,000 chat requests per minute. What protects your gateway and LLM bill?"*

### Layer 1 – Short Answer
> "Three layers absorb the blast: `TokenBucketRateLimitMiddleware` caps the tenant's API key at 100 burst / 10 req/s in Redis Lua and sheds excess traffic with HTTP 429 in ~1ms before DB or LLM calls execute; if they repeat the same prompt, `RedisSemanticCache` intercepts hits at $0 token cost; and Nginx protects overall socket saturation."

### Layer 2 – Remaining Vulnerabilities & Fixes
- **Gap 1 (Request Count vs. Token Weight):** Currently `TokenBucketRateLimitMiddleware` passes `requested=1` regardless of prompt size. A tenant sending 10 req/s with 8,000-token prompts still consumes 4.8M tokens/min! **Fix:** Compute input token weight with `tiktoken` (or byte-length estimate) and pass `requested = token_cost_units` into `limiter.check()`.
- **Gap 2 (No Monthly Dollar Hard Cap):** `TokenAnalyticsTracker` tracks `analytics:tenant:{id}:dollars_spent`, so adding a pre-flight budget check in `chat.py` (raising RFC 7807 `402 Payment Required` when spend exceeds quota) completes financial protection.
- **Gap 3 (Missing `X-API-Key`):** If a caller omits `X-API-Key`, rate limiting falls back to `ip:{client.host}`—in shared corporate NATs or behind misconfigured proxies without trusting `X-Forwarded-For`, multiple tenants on one IP could share one bucket, or rotating IPs could bypass IP buckets. Production deployment should require valid authenticated API keys (`401` on missing/invalid key).

---

## Scenario 6: Intermittent `asyncpg.InterfaceError: another operation is in progress`

**Interviewer:** *"A teammate tries to speed up `SearchRepository` by wrapping `search_dense` and `search_sparse` in `asyncio.gather()`, and production immediately starts throwing `InterfaceError: cannot perform operation: another operation is in progress`. Why?"*

### Answer
`AsyncSession` holds a single `asyncpg` TCP socket that operates strictly as a half-duplex request-response channel. Calling `session.execute()` concurrently from two coroutines attempts to interleave wire frames on one connection, which `asyncpg` rejects immediately. Either revert to sequential execution on the single session (as SentinelRAG does in `app/repositories/search.py`, keeping total query time low while using 1 pool slot), or check out two independent `AsyncSession` instances from `async_session_factory` if queries genuinely need parallel execution.

---

## Scenario 7: Stale or Wrong Answers Returned After Document Updates

**Interviewer:** *"A tenant updates their HR parental leave policy PDF and re-ingests chunks via `/documents/batch`, yet users keep receiving the old 4-week policy answer. Why and how do you fix it?"*

### Answer
1. **Why:** `RedisSemanticCache` still holds the prior answer under `cache:{tenant_id}:{sha256}` for up to 24 hours (`86400s` TTL). Re-ingesting documents into PostgreSQL does not automatically invalidate Redis.
2. **Immediate Fix:** Call `POST /api/v1/cache/invalidate` with `{"tenant_id": "..."}` right after document ingestion completes (`RedisSemanticCache.invalidate_tenant` deletes all keys tracked in `cache:{tenant_id}:keys` atomically).
3. **Long-Term Improvement:** Also support document-level or `document_name` cache tagging (storing `source_doc_ids` in cache metadata so updating one policy PDF invalidates only dependent cache keys instead of flushing the tenant's entire cache), and audit whether `SEMANTIC_CACHE_THRESHOLD` (`0.08`) was widened too far if distinct topics collided.

---
**Next:** [09-design-decisions-and-tradeoffs.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md)

