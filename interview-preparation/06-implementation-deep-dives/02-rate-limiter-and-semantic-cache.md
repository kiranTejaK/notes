# Implementation Deep Dive 2: Rate Limiter (Lua) & Two-Tier Semantic Cache

**Covered Modules (Verified in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md)):**
`app/core/rate_limiter.py` | `app/services/cache.py` | `app/schemas/cache.py` | `app/api/v1/cache.py`

---

## 1. Distributed Token-Bucket Rate Limiter (`app/core/rate_limiter.py`)

### Key Classes & Symbols
- `TOKEN_BUCKET_LUA_SCRIPT`: Single atomic Lua script executed inside Redis.
- `RateLimitResult` (`@dataclass(slots=True)`): `allowed`, `remaining`, `retry_after`, `limit`.
- `TokenBucketLimiter`: Manages `SCRIPT LOAD` caching (`self._script_sha`), `EVALSHA` execution, `NoScriptError` recovery, and fail-open fallback.
- `TokenBucketRateLimitMiddleware(BaseHTTPMiddleware)`: Enforces quotas before route handlers execute.

### Annotated Lua Script Walkthrough
```lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])

-- 1. Redis server clock prevents worker NTP skew across pods
local time_arr = redis.call('TIME')
local now = tonumber(time_arr[1]) + (tonumber(time_arr[2]) / 1000000)

-- 2. Lazy refill from stored bucket hash
local current_data = redis.call('HMGET', key, 'tokens', 'last_updated')
local tokens = tonumber(current_data[1])
local last_updated = tonumber(current_data[2])

if tokens == nil or last_updated == nil then
    tokens = capacity
    last_updated = now
else
    local delta = math.max(0, now - last_updated) -- guards backward clock step
    tokens = math.min(capacity, tokens + (delta * refill_rate))
    last_updated = now
end

-- 3. Conditional allow/deny & Retry-After calculation
local allowed = 0
local retry_after = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
else
    local needed = requested - tokens
    retry_after = math.ceil(needed / refill_rate)
end

-- 4. Persist & auto-expire inactive keys (min 60s or 2x full-refill window)
redis.call('HSET', key, 'tokens', tokens, 'last_updated', last_updated)
local ttl = math.ceil((capacity / refill_rate) * 2)
redis.call('EXPIRE', key, math.max(60, ttl))

return {allowed, math.floor(tokens), retry_after, capacity}
```

### Non-Obvious Implementation Details
1. **`EVALSHA` + `NoScriptError` Resilience:** `TokenBucketLimiter._get_script_sha` loads the script once via `SCRIPT LOAD`. If Redis restarts or runs `SCRIPT FLUSH`, `evalsha` raises `NoScriptError`; `check()` resets `self._script_sha = None` and immediately retries with `client.eval(...)` without dropping the user's request.
2. **Identifier Precedence:** Middleware checks `X-API-Key` (`ratelimit:key:{api_key}`) and falls back to client IP (`ratelimit:ip:{host}`). Exempts `/healthz`, `/`, `/docs`, `/redoc`, `/openapi.json`.
3. **Fail-Open Safeguard:** If `self.redis is None` or Redis raises any exception, `check()` logs an error and returns `allowed=True` so Redis unavailability doesn't block all traffic.
4. **Config vs. Signature Default:** `TokenBucketLimiter.check` signature defaults to `(capacity=60, refill_rate_per_sec=1.0)`, while `app/main.py` wires `settings` defaults (`capacity=100, refill_rate=10/s`).

---

## 2. Two-Tier Redis Semantic Cache (`app/services/cache.py`)

### Key Classes & Functions
- `SemanticLookupResult` (`@dataclass(slots=True)`): Captures `status` (`"CACHE_HIT"` / `"CACHE_MISS"`), `similarity`, `distance`, `threshold`, `cached_query`, `cached_response`, `matched_key`, `lookup_latency_ms`, and `metadata`.
- `serialize_embedding` / `deserialize_embedding`: Packs `list[float]` into raw 32-bit IEEE 754 binary via `struct.pack(f"{len(vector)}f", *vector)` (`6,144` bytes for 1536d) and Base64-encodes to `8,192` ASCII characters (UTF-8 safe for `decode_responses=True`).
- `compute_cosine_similarity(vec_a, vec_b)`: Pure-Python dot product divided by $\|\mathbf{a}\|\|\mathbf{b}\|$, clamped to `[-1.0, 1.0]`, returning `0.0` on length mismatch or zero norm.
- `RedisSemanticCache`:
  - **Key Schema:** `cache:{tenant_id}:{sha256_hash}` (Redis Hash storing `query`, `embedding`, `response`, `metadata`, `created_at`) + `cache:{tenant_id}:keys` (Redis Set tracking active tenant keys).

### Two-Stage Lookup Mechanics (`get_or_lookup`)
1. **Stage 1 — Exact-Match Fast Path:**
   - Normalizes `query.strip().lower()`, computes `SHA-256`, and calls `await self.redis.hgetall(exact_key)`.
   - On hit (`"response" in exact_data`), returns immediately with `similarity=1.0, distance=0.0`—zero vector decoding or dot products.
2. **Stage 2 — Pipelined Semantic Scan + Lazy Eviction:**
   - Fetches tenant keys via `await self.redis.smembers(index_key)`.
   - Uses `self.redis.pipeline(transaction=False)` to batch `HGETALL` for every key in one network RTT (`transaction=False` avoids unnecessary `MULTI/EXEC` locking overhead for reads).
   - **Self-Healing Cleanup:** When an individual cache key expires after `86400s`, it remains in `cache:{tenant_id}:keys`. Any empty hash (`not entry`) or corrupt embedding is collected in `expired_keys` and pruned via `await self.redis.srem(index_key, *expired_keys)`.
   - Computes `best_distance = round(1.0 - best_similarity, 5)`. Hits if `best_distance <= effective_threshold` (default `0.08`, i.e., similarity $\ge 0.92$).

### Transactional Write & Tenant Invalidation
- **`set()`:** Uses `self.redis.pipeline(transaction=True)` to execute `HSET`, `EXPIRE(cache_key, ttl)`, `SADD(index_key, cache_key)`, and `EXPIRE(index_key, ttl * 2)` atomically.
- **`invalidate_tenant()`:** Fetches `smembers(index_key)` and deletes every tenant cache hash plus `index_key` in one `transaction=True` pipeline, returning `evicted_count`.

### Cache Endpoints (`app/api/v1/cache.py`) & Edge Cases
- `POST /api/v1/cache/evaluate`: Tests hit/miss latency; optionally stores response on miss (`store_if_miss=True`).
- `POST /api/v1/cache/set`: Explicit cache warm-up (`201 Created`).
- `POST /api/v1/cache/invalidate`: Purges all entries for a `tenant_id`.
- **Edge Cases to Watch:**
  - *Cross-Architecture Endianness:* `struct.pack(f"{len(vector)}f", ...)` uses native endianness; specifying `<` (little-endian) prevents byte-order bugs if ARM64 and x86 workers share Redis.
  - *`query_vector=None` from `chat.py`:* `get_or_lookup` expects `list[float]` for Stage 2; if `None` is passed when `smembers` is non-empty, `compute_cosine_similarity` raises `TypeError` which is safely caught and logged as `semantic_cache_lookup_failed` $\to$ `CACHE_MISS`.

---
**Next:** [03-hybrid-search-and-rrf.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md)

