# Technology Deep Dive 3: Redis 7.2, Atomic Lua Scripting & Two-Tier Semantic Caching

## 1. Fundamental Concepts & Internals

### Redis Single-Threaded Execution & Lua Atomicity
- **Core Loop:** Redis executes commands on a **single main event-loop thread** (`epoll`/`kqueue`). Because commands run sequentially without OS thread preemptive switching, every single Redis command—and every **Lua script**—is atomic and isolated.
- **`EVAL` vs. `EVALSHA`:** `EVAL` resends the entire Lua source string over the network on every request. `SCRIPT LOAD` compiles and caches bytecode inside Redis RAM keyed by its 40-char SHA-1 digest; `EVALSHA` sends only the SHA-1 hash. Because Redis script caches live in volatile RAM, `SCRIPT FLUSH` or a failover clears them, raising `NoScriptError`—so clients must fall back transparently to `EVAL`.
- **Latency Rule:** Because a Lua script blocks *all* other Redis clients while running (guarded by `lua-time-limit`, default 5s), Lua scripts must execute in $O(1)$ microseconds—never iterate huge keyspaces inside Lua.

### Memory Encodings & Eviction (`volatile-lru`)
- **Strings vs. Hashes (`listpack` vs. `hashtable`):** Small hashes ($\le 512$ fields, values $\le 64$ bytes) pack into a contiguous `listpack` array with zero per-field pointers. However, in SentinelRAG's semantic cache, a base64-encoded 1536-dim float vector is **8,192 bytes**, exceeding `hash-max-listpack-value` (64B), so Redis promotes each cache hash to `OBJ_ENCODING_HT` (`dict`).
- **`volatile-lru` (`maxmemory 512mb` in `docker-compose.prod.yml`):** Redis approximates LRU by sampling `maxmemory-samples` (default 5) keys and evicting the oldest idle time. `volatile-lru` **only** evicts keys that have an explicit `EXPIRE` TTL, protecting non-expiring operational data (like analytics counters `analytics:*`).

---

## 2. Why Selected for SentinelRAG
- **Shared State Across Workers:** Multi-worker Uvicorn/Docker containers cannot share in-memory Python dicts or local FAISS indices without cache fragmentation ($1/N$ hit rate) and $N\times$ RAM duplication.
- **Vanilla Redis Portability (No Custom Modules):** `RedisSemanticCache` stores packed embeddings in standard Redis 7 hashes rather than requiring Redis Stack (`RediSearch` C module), so it deploys cleanly on any managed Redis (ElastiCache, Azure Cache, vanilla `redis:7.2-alpine`).
- **One Round-Trip Rate Limiting:** Lua handles read-time-refill-check-decrement-expire atomically without `Redlock` overhead or `MULTI/EXEC` limitations.

---

## 3. Project Implementation & Configuration Details

### A. Connection Pool (`app/core/redis.py`)
```python
self.pool = aioredis.ConnectionPool.from_url(
    settings.REDIS_URL,
    max_connections=50,
    decode_responses=True,   # Forces UTF-8 strings on all reads
    socket_timeout=0.5,      # 500ms hard cap prevents ASGI stalls
    socket_connect_timeout=0.5,
)
```
- **Crucial Consequence of `decode_responses=True`:** Raw `struct.pack` bytes are arbitrary binary values that fail UTF-8 decoding—hence `serialize_embedding` base64-encodes the `6,144` bytes into `8,192` safe ASCII characters.

### B. Distributed Token Bucket (`app/core/rate_limiter.py`)
- **Key:** `ratelimit:key:{X-API-Key}` or `ratelimit:ip:{client.host}`.
- **Lua Script Mechanics:**
  1. Reads authoritative clock inside Redis: `redis.call('TIME')` (`sec + usec / 1e6`) — eliminates NTP clock drift across container hosts.
  2. Reads `HMGET key tokens last_updated`. Initializes `tokens = capacity` (`100`) if missing.
  3. Computes `delta = math.max(0, now - last_updated)` (guards against backward clock steps) and refills `math.min(capacity, tokens + delta * refill_rate)`.
  4. If `tokens >= requested`, decrements and sets `allowed = 1`; else computes `retry_after = math.ceil((requested - tokens) / refill_rate)`.
  5. Writes `HSET` and `EXPIRE key math.max(60, math.ceil((capacity / refill_rate) * 2))` to prevent abandoned bucket memory leaks.
- **Fail-Open Policy:** Any Redis connection/timeout error logs `rate_limit_redis_evaluation_error` and returns `allowed=True` so Redis hiccups never take down the gateway (paired with Nginx outer IP limits in production).

### C. Two-Tier Semantic Cache (`app/services/cache.py`)
1. **Tier 1 — $O(1)$ Exact Fast Path:**
   - `compute_query_hash(query)` normalizes `query.strip().lower()` and computes **SHA-256** (`cache:{tenant_id}:{sha256}`).
   - `await self.redis.hgetall(exact_key)` returns immediately on hit (`similarity=1.0, distance=0.0`).
2. **Tier 2 — Tenant-Isolated Vector Scan:**
   - Fetches tenant keys via `SMEMBERS cache:{tenant_id}:keys`, runs a non-transactional pipeline (`pipeline(transaction=False)`) of `HGETALL`s, decodes embeddings via `base64` + `struct.unpack`, computes cosine similarity, and checks `best_distance = round(1.0 - best_similarity, 5) <= 0.08` ($\text{similarity} \ge 0.92$).
   - **IEEE 754 Packing Savings:** 1536 floats as JSON text take ~28–35 KB and heavy string parsing; `struct.pack(f"{len(vec)}f", *vec)` takes `6,144` bytes (`8,192` chars in Base64, ~70–75% smaller and microsecond unpack).
   - **Ghost Key Lazy Eviction:** Individual `cache:{tenant_id}:{hash}` entries expire after `86,400s` (24h), leaving stale key strings inside the Redis Set `cache:{tenant_id}:keys`. When `pipeline.hgetall()` returns empty dicts (or corrupt entries), `get_or_lookup` runs `await self.redis.srem(index_key, *expired_keys)`—self-healing without a cron job.
   - **Tenant Invalidation (`POST /api/v1/cache/invalidate`):** Uses a `transaction=True` pipeline deleting all member keys in `cache:{tenant_id}:keys` plus the index set itself.

### D. Atomic Token & Spend Counters (`app/services/token_analytics.py`)
- Uses `redis.pipeline()` with atomic `INCRBY` and `INCRBYFLOAT` (`analytics:tenant:{id}:dollars_saved`, `dollars_spent`, `prompt_tokens`) instead of Python read-modify-write (`GET` $\to$ add $\to$ `SET`), eliminating lost updates across concurrent workers.

---

## 4. Practical Problems, Trade-Offs & Scaling Path
1. **Tier 2 Linear Scan Bottleneck ($O(N \cdot D)$):** Fetching and looping over candidate hashes in Python stays fast ($\le 15\text{ms}$) up to a few thousand active keys per tenant, but degrades beyond ~5,000–10,000 entries/tenant (network bandwidth + pure-Python loop holding the GIL).
   - *Upgrade Path:* Migrate Tier 2 to **RediSearch Vector Similarity (`FT.CREATE` / `FT.SEARCH ... KNN`)** in C/HNSW, or vectorize the Python loop with NumPy (`BLAS` releases the GIL).
2. **Cache Stampede (Thundering Herd) on Cache Misses:** Currently, if 100 concurrent requests ask the exact same novel question simultaneously, all 100 miss and call the LLM (`HSET` is idempotent so state stays intact, but tokens are wasted).
   - *Upgrade Path:* Add a short-lived Redis singleflight mutex (`SET cache:lock:{hash} 1 NX EX 15`) or probabilistic early refresh (`XFetch`).
3. **Copy-on-Write (`BGSAVE`) OOM Risk in Docker:** If Redis uses 350MB inside a 512MB cgroup and forks for `BGSAVE` or AOF rewrite under heavy writes, Linux CoW page duplication can trigger OOM kill (`137`). Keep `maxmemory` $\le 50\text{--}65\%$ of container memory limit.
4. **Endianness Note:** `cache.py` calls `struct.pack(f"{len(vector)}f", *vector)` using host native byte order. Explicit `<` little-endian (`struct.pack(f"<{len(vector)}f", *vector)`) guarantees portability across x86 and ARM64 workers sharing one Redis instance.

---

## 5. Quick Interview Questions
1. *Why can't Redis `MULTI/EXEC` replace a Lua script for Token Bucket rate limiting?*
2. *Why does SentinelRAG base64-encode `struct.pack` floats, and what breaks if you remove Base64?*
3. *How does `RedisSemanticCache` avoid unbounded growth of the `cache:{tenant_id}:keys` set when keys TTL-expire?*
4. *At what point does `SMEMBERS` + pipelined Python cosine scan break down, and how do you evolve it?*

