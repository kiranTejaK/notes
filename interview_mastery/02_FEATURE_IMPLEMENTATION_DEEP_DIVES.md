# SentinelRAG: Feature Implementation Deep Dives

> **Target Audience:** Python Backend Developers (2–4 Years Experience) preparing for Senior / Mid-Senior Technical Interviews.  
> **Workspace Root:** [c:/Users/kiran/Desktop/sentinel-rag](file:///c:/Users/kiran/Desktop/sentinel-rag)

---

## Feature 1: Two-Stage Redis Semantic Cache with Vector Similarity

### A. Purpose
- **Problem Solved:** Repetitive questions to frontier LLMs incur cold generation latency (800ms–3000ms) and burn token budgets (\$0.15–\$10.00 / 1M tokens). Standard HTTP or key-value caches only match identical strings. If a user asks *"How do I reset my password?"* and another asks *"Where can I change my password?"*, an exact-match cache misses both times.
- **Why Needed in SentinelRAG:** SentinelRAG serves high-volume enterprise traffic where 40–60% of queries share identical semantic intent. Caching responses semantically slashes operational cloud spend and drops p95 response times to $< 20$ms.
- **What Happens Without It:** Every query hits upstream inference APIs, incurring linear cost with request volume, degrading user experience during provider latency spikes, and rapidly exhausting provider Rate Limits (HTTP 429).

### B. Core Concept
Semantic caching evaluates whether a new query is *conceptually equivalent* to a previously answered query by comparing high-dimensional mathematical representations (embeddings). If the angle between the two vectors in high-dimensional space is near zero (cosine distance $\le 0.08$), the queries have identical semantic intent, and the previous response can be returned immediately.

---

### C. Actual Implementation in SentinelRAG
- **File:** [app/services/cache.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py)
- **Primary Class:** [RedisSemanticCache](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L79-L336)
- **Key Functions:**
  - `compute_query_hash(query: str) -> str`: Normalizes string (strip, lowercase) and produces SHA-256 digest.
  - `get_or_lookup(tenant_id, query, query_vector, threshold) -> SemanticLookupResult`
  - `set(tenant_id, query, query_vector, response, metadata, ttl_seconds) -> str`
  - `serialize_embedding(vector: list[float]) -> str`: Serializes float vector into base64-encoded IEEE 754 32-bit binary floats via Python's `struct.pack`.
  - `deserialize_embedding(serialized: str) -> list[float]`: Unpacks base64 binary back to `list[float]`.
  - `compute_cosine_similarity(vec_a, vec_b) -> float`: Computes dot product divided by product of Euclidean norms.

```python
# Code Excerpt: app/services/cache.py (Lines 115-139, 201-227)
# 1. Exact-match Fast Path (O(1))
exact_data = await self.redis.hgetall(exact_key)
if exact_data and "response" in exact_data:
    return SemanticLookupResult(
        status="CACHE_HIT",
        similarity=1.0,
        distance=0.0,
        cached_response=exact_data.get("response"),
        matched_key=exact_key,
        ...
    )

# 2. Semantic Similarity Scan across Tenant Keys
# Evaluates threshold: distance <= 0.08
best_distance = round(1.0 - best_similarity, 5)
if best_distance <= effective_threshold:
    return SemanticLookupResult(
        status="CACHE_HIT",
        similarity=best_similarity,
        distance=best_distance,
        cached_response=best_entry.get("response"),
        ...
    )
```

#### Execution Details & Non-Obvious Logic
1. **Two-Stage Fast Path:** Exact string matching is evaluated first. If an SHA-256 hash hit occurs, execution completes in $< 2$ms without reading vector buffers or calculating floating-point dot products.
2. **Compact IEEE 754 Packing:** Floats are not stored as JSON arrays of strings (which consume ~12KB per 1536d vector). Instead, `struct.pack(f"{len(vector)}f", *vector)` packs them into 6144 raw bytes, base64-encoded to ~8KB, ensuring fast deserialization and UTF-8 safety with `redis.asyncio`'s `decode_responses=True`.
3. **Lazy Eviction of Dead Keys:** When candidate keys from `SMEMBERS cache:{tenant_id}:keys` have expired their TTL in Redis, `pipeline.hgetall()` returns empty dicts. The service detects this and fires an asynchronous `srem` to prune dead key names from the tenant tracking set, preventing set unbounded memory growth.

### D. Why It Was Implemented This Way
- **Alternative 1: In-Memory Python Dict (LRU):** Fails in multi-process/multi-container deployments. If 4 Uvicorn workers run behind Nginx, each maintains a separate cache, causing duplicate cache misses and excessive RAM consumption.
- **Alternative 2: Dedicated Vector DB (Pinecone/Milvus) for Caching:** Adds 40–80ms of network overhead and substantial infrastructure cost. Redis gives sub-millisecond in-memory lookups.
- **Threshold Choice ($\le 0.08$ distance / $\ge 0.92$ similarity):** Empirical benchmarking across embedding models (like OpenAI `text-embedding-3-small`) shows that cosine distance $\le 0.08$ strictly separates paraphrased identical intents from subtle semantic differences (e.g., *"How to delete account?"* vs. *"How to suspend account?"*).

### E. Interview Follow-Up Questions & Production Answers
- **Q: What happens if two concurrent requests miss the cache with the same query?**  
  *Answer:* Both will execute hybrid retrieval and LLM generation (cache stampede). Once completed, both will call `cache_service.set()`. Because `set()` uses Redis `HSET` and transactions, the write is idempotent; the second write cleanly overwrites the first without corrupting state.
- **Q: How do you handle cache invalidation when underlying enterprise documents change?**  
  *Answer:* We implemented tenant-scoped invalidation: [invalidate_tenant(tenant_id)](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L315-L336). When an ingestion job updates documents for a tenant, it issues `POST /api/v1/cache/invalidate`, which reads `cache:{tenant_id}:keys` and atomically deletes all cached responses in that namespace.

### F. Simplified Illustrative Implementation
```python
# Simplified Teaching Example: Core Semantic Cache Mechanism
import math

class SimpleSemanticCache:
    def __init__(self, threshold: float = 0.08):
        self.cache: dict[str, tuple[list[float], str]] = {}
        self.threshold = threshold

    def _cosine_dist(self, u: list[float], v: list[float]) -> float:
        dot = sum(a * b for a, b in zip(u, v))
        norm_u = math.sqrt(sum(a * a for a in u))
        norm_v = math.sqrt(sum(b * b for b in v))
        sim = dot / (norm_u * norm_v) if norm_u and norm_v else 0.0
        return 1.0 - max(-1.0, min(1.0, sim))

    def get(self, query_vec: list[float]) -> str | None:
        best_dist = float("inf")
        best_ans = None
        for vec, ans in self.cache.values():
            dist = self._cosine_dist(query_vec, vec)
            if dist < best_dist:
                best_dist = dist
                best_ans = ans
        return best_ans if best_dist <= self.threshold else None

    def put(self, key: str, vec: list[float], response: str) -> None:
        self.cache[key] = (vec, response)
```

### G. Modification Exercise
- **Task:** Make the cache threshold dynamically configurable per API request instead of relying solely on server environment settings.
- **Files to Change:** [app/schemas/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/schemas/chat.py#L42), [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py#L88).
- **Approach:** Add `semantic_threshold: float | None = Field(default=None, ge=0.01, le=0.50)` to `ChatCompletionRequest`. Pass `threshold=payload.semantic_threshold` to `cache_service.get_or_lookup()`.

---

## Feature 2: Single-Store Hybrid Retrieval (pgvector HNSW + GIN tsvector)

### A. Purpose
- **Problem Solved:** Semantic dense vector search is outstanding at conceptual understanding, but notoriously weak at finding exact alphanumeric identifiers (part numbers, error codes like `ERR-40412`, SKUs, and exact names). Conversely, keyword search (BM25/FTS) is brittle when users use synonyms.
- **Why Needed in SentinelRAG:** Enterprise documentation requires both conceptual understanding and exact keyword recall.
- **What Happens Without It:** A purely vector-based system hallucinates or returns irrelevant documents when an engineer searches for a specific technical constant or acronym.

### B. Core Concept
Hybrid retrieval queries two distinct representation spaces for the same query:
1. **Dense Semantic Space:** Converts text to high-dimensional floating-point vectors (1536 dimensions) and performs nearest-neighbor vector search.
2. **Sparse Lexical Space:** Analyzes word stems, token frequencies, and inverse document frequencies via full-text search.

---

### C. Actual Implementation in SentinelRAG
- **Model File:** [app/models/document.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/models/document.py#L16-L84)
- **Repository File:** [app/repositories/search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L34-L152)
- **Model Definition:**
  ```python
  # Code Excerpt: app/models/document.py
  embedding: Mapped[list[float]] = mapped_column(Vector(1536), nullable=False)
  search_vector: Mapped[Any] = mapped_column(
      TSVECTOR,
      Computed("to_tsvector('english', chunk_text)", persisted=True),
      nullable=True,
  )
  __table_args__ = (
      Index("ix_document_chunks_search_vector", "search_vector", postgresql_using="gin"),
      Index(
          "ix_document_chunks_embedding_hnsw",
          "embedding",
          postgresql_using="hnsw",
          postgresql_with={"m": 16, "ef_construction": 64},
          postgresql_ops={"embedding": "vector_cosine_ops"},
      ),
  )
  ```
- **Repository Execution:**
  - `search_dense`: Uses `DocumentChunk.embedding.cosine_distance(query_vector)` which generates the native SQL `<=>` operator.
  - `search_sparse`: Uses `func.ts_rank_cd(DocumentChunk.search_vector, ts_query)` with `DocumentChunk.search_vector.op("@@")(ts_query)`.
  - `search_hybrid_candidates`: Queries dense first, then sparse sequentially on the `AsyncSession`.

```python
# Code Excerpt: app/repositories/search.py (Lines 57-64, 104-115)
# Dense Query:
distance_expr = DocumentChunk.embedding.cosine_distance(query_vector).label("distance")
stmt = (
    select(DocumentChunk, distance_expr)
    .where(DocumentChunk.tenant_id == tenant_id)
    .order_by(distance_expr.asc())
    .limit(limit)
)

# Sparse Query:
ts_query = func.websearch_to_tsquery("english", sanitized_query)
rank_expr = func.ts_rank_cd(DocumentChunk.search_vector, ts_query).label("rank_score")
stmt = (
    select(DocumentChunk, rank_expr)
    .where(
        DocumentChunk.tenant_id == tenant_id,
        DocumentChunk.search_vector.op("@@")(ts_query),
    )
    .order_by(rank_expr.desc())
    .limit(limit)
)
```

### D. Why It Was Implemented This Way
- **Single-Store Rationale:** Storing relational metadata, JSONB tags, dense vectors, and lexical tsvectors in the exact same row in PostgreSQL 16 eliminates the dual-system problem. Transactions are ACID-compliant: an update to a document chunk updates both its vector and its text vector atomically.
- **Why HNSW over IVFFlat?** IVFFlat requires an initial training step and clustering centers, degrading search quality as new documents are inserted. HNSW (Hierarchical Navigable Small World) provides superior query recall ($> 98\%$) and consistent sub-50ms latency across 500k+ chunks without manual re-indexing.
- **Why Sequential Session Execution?** In [app/repositories/search.py#L140](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L140), dense and sparse queries are executed sequentially:
  ```python
  # Sequential execution to avoid asyncpg connection socket collision
  dense_candidates = await self.search_dense(...)
  sparse_candidates = await self.search_sparse(...)
  ```
  `asyncpg` uses a single underlying TCP socket per connection. Attempting `asyncio.gather(self.search_dense(), self.search_sparse())` on the *same* session crashes with `InterfaceError: cannot perform operation: another operation is in progress`.

### E. Interview Follow-Up Questions & Production Answers
- **Q: Why use `ts_rank_cd` instead of standard `ts_rank`?**  
  *Answer:* Standard `ts_rank` only counts lexeme frequency. `ts_rank_cd` computes *cover density*: it factors in how close the query words appear to one another within the document text. Proximity is a critical relevance signal for human search queries.
- **Q: How does HNSW handle multi-tenant filtering?**  
  *Answer:* In PostgreSQL 16 `pgvector`, HNSW indexes support iterative index scans with `WHERE tenant_id = :id`. It traverses the graph and dynamically filters nodes by the B-Tree indexed tenant ID.

### F. Simplified Illustrative Implementation
```python
# Simplified SQL representation of SentinelRAG's dual candidate retrieval
SQL_HYBRID_QUERY = """
-- 1. Dense candidates
SELECT id, chunk_text, embedding <=> :query_vector AS distance
FROM document_chunks
WHERE tenant_id = :tenant_id
ORDER BY distance ASC LIMIT 50;

-- 2. Sparse candidates
SELECT id, chunk_text, ts_rank_cd(search_vector, websearch_to_tsquery('english', :query_text)) AS rank
FROM document_chunks
WHERE tenant_id = :tenant_id AND search_vector @@ websearch_to_tsquery('english', :query_text)
ORDER BY rank DESC LIMIT 50;
"""
```

### G. Modification Exercise
- **Task:** Add JSONB metadata filtering (e.g., filter by department or document tag) to the dense search query.
- **File to Change:** [app/repositories/search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L59-L64).
- **Approach:** Add `metadata_filter: dict[str, Any] | None = None` parameter. Add condition:
  `if metadata_filter: stmt = stmt.where(DocumentChunk.metadata_json.contains(metadata_filter))`. This leverages the existing GIN index on `metadata_json`.

---

## Feature 3: Mathematical Reciprocal Rank Fusion (RRF, $k=60$)

### A. Purpose
- **Problem Solved:** When merging candidates from dense vector search and sparse full-text search, raw scores cannot be compared. Vector cosine distance is bounded between $[0.0, 2.0]$ (where smaller is better), while full-text `ts_rank_cd` is an unbounded float $\ge 0.0$ (where larger is better). Linear score combinations (e.g., $\alpha \cdot \text{dense} + (1-\alpha) \cdot \text{sparse}$) fail because score distributions vary drastically across queries.
- **Why Needed in SentinelRAG:** SentinelRAG requires a robust, zero-calibration method to merge candidate pools without manual tuning.
- **What Happens Without It:** One retrieval modality dominates the results, completely drowning out the other or returning poorly sorted chunks to the LLM.

### B. Core Concept
Reciprocal Rank Fusion (Cormack, Clarke, and Büttcher, 2009) evaluates only the *rank position* of documents within each list, completely discarding the raw scores. The formula is:
$$\text{RRF\_Score}(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
Where $M$ is the set of modalities (dense and sparse), $r_m(d)$ is the 1-based rank of document $d$, and $k$ is a smoothing constant (default: 60). Documents appearing near the top of *both* lists receive the highest fused score.

---

### C. Actual Implementation in SentinelRAG
- **File:** [app/services/rrf.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/rrf.py)
- **Key Functions:**
  - `reciprocal_rank_fusion(rankings: Sequence[Sequence[T]], k: int = 60) -> list[tuple[T, float]]`
  - `fuse_hybrid_candidates(dense_candidates, sparse_candidates, k=60, top_k=None) -> list[FusedSearchResult]`

```python
# Code Excerpt: app/services/rrf.py (Lines 98-135)
for cand in dense_candidates:
    c_id = cand.chunk.id
    chunk_map[c_id] = cand.chunk
    dense_info[c_id] = (cand.rank, cand.similarity_score)
    fused_scores[c_id] = fused_scores.get(c_id, 0.0) + (1.0 / (k + cand.rank))

for cand in sparse_candidates:
    c_id = cand.chunk.id
    chunk_map[c_id] = cand.chunk
    sparse_info[c_id] = (cand.rank, cand.lexical_score)
    fused_scores[c_id] = fused_scores.get(c_id, 0.0) + (1.0 / (k + cand.rank))

# Deterministic sorting: primary=fused_score, tie-break=dense similarity
fused_results.sort(
    key=lambda r: (
        r.fused_score,
        r.dense_score if r.dense_score is not None else -1.0,
        r.sparse_score if r.sparse_score is not None else -1.0,
    ),
    reverse=True,
)
```

### D. Why It Was Implemented This Way
- **Zero Hyperparameter Tuning:** Does not require min-max normalization or manual weight calibrations that break when switching embedding models.
- **Why $k=60$?** Cormack et al. proved empirically that $k \approx 60$ acts as a dampener. If $k=1$, rank 1 gives $\frac{1}{2} = 0.5$ while rank 2 gives $\frac{1}{3} = 0.33$ (a huge 34% drop). With $k=60$, rank 1 gives $\frac{1}{61} \approx 0.01639$ and rank 2 gives $\frac{1}{62} \approx 0.01612$, preventing a single lucky top-1 match from overwhelming a document that ranked 2nd in both dense and sparse.
- **Deterministic Tie-Breaking:** When two chunks have equal fused scores, SentinelRAG breaks the tie using `dense_score` (semantic similarity), followed by `sparse_score`.

### E. Interview Follow-Up Questions & Production Answers
- **Q: What is the computational complexity of RRF?**  
  *Answer:* $O(N_1 + N_2 + U \log U)$, where $N_1$ and $N_2$ are the candidate pool sizes (e.g., 50 each), and $U \le 100$ is the number of unique documents. Because candidate pools are small, RRF executes in $< 1$ millisecond in pure Python without NumPy overhead.
- **Q: How would RRF behave if a document only appears in one list?**  
  *Answer:* It still receives a score from that list: $\frac{1}{60 + r}$. However, a document that appears in both lists (e.g., rank 10 in dense and rank 15 in sparse) will score $\frac{1}{70} + \frac{1}{75} \approx 0.01428 + 0.01333 = 0.02761$, easily beating a document that ranked 1st in only one list ($\frac{1}{61} \approx 0.01639$).

### F. Simplified Illustrative Implementation
```python
# Simplified RRF Implementation
def simple_rrf(list_a: list[str], list_b: list[str], k: int = 60) -> list[tuple[str, float]]:
    scores: dict[str, float] = {}
    for rank, item in enumerate(list_a, start=1):
        scores[item] = scores.get(item, 0.0) + (1.0 / (k + rank))
    for rank, item in enumerate(list_b, start=1):
        scores[item] = scores.get(item, 0.0) + (1.0 / (k + rank))
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### G. Modification Exercise
- **Task:** Implement weighted RRF, where dense vector search receives a $1.5\times$ multiplier relative to sparse search.
- **File to Change:** [app/services/rrf.py#L102-L108](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/rrf.py#L102-L108).
- **Approach:** Add `dense_weight: float = 1.0` and `sparse_weight: float = 1.0`. Multiply:
  `fused_scores[c_id] += dense_weight * (1.0 / (k + cand.rank))`.

---

## Feature 4: Coroutine-Safe State-Machine Circuit Breaker

### A. Purpose
- **Problem Solved:** When an upstream LLM provider suffers an outage or severe degradation, requests pile up waiting for timeouts (e.g., 10s–30s). This exhausts ASGI worker threads and connection pools, causing the entire gateway to freeze (cascading failure).
- **Why Needed in SentinelRAG:** SentinelRAG's primary model is Groq. When Groq hits rate limits or undergoes maintenance, SentinelRAG must immediately protect the gateway and redirect traffic to backup models.
- **What Happens Without It:** Gateway response latency jumps from 200ms to 30s per request during third-party incidents, crashing client applications.

### B. Core Concept
The Circuit Breaker pattern (Michael Nygard, *Release It!*) acts like an electrical fuse with three operational states:
1. **`CLOSED`:** Normal operation. Requests flow to the primary provider. Consecutive failures are tracked.
2. **`OPEN`:** The error threshold was reached (5 errors). The circuit trips. All requests immediately fail fast in 0ms without making network calls.
3. **`HALF_OPEN`:** The recovery cooldown (60s) has elapsed. The circuit permits a single canary request to test upstream health. If it succeeds, the circuit recovers to `CLOSED`. If it fails, the circuit re-trips to `OPEN`.

---

### C. Actual Implementation in SentinelRAG
- **File:** [app/core/circuit_breaker.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py)
- **Primary Class:** [CircuitBreaker](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py#L41-L184)
- **Key Methods:**
  - `can_execute() -> bool`: Evaluates state and checks recovery cooldown elapsed.
  - `record_success()`: Clears failure counts and resets `HALF_OPEN` $\to$ `CLOSED`.
  - `record_failure()`: Increments failure count; trips to `OPEN` if $\ge 5$ failures.

```python
# Code Excerpt: app/core/circuit_breaker.py (Lines 85-106, 121-148)
async def can_execute(self) -> bool:
    async with self._lock:
        if self._state == CircuitState.CLOSED:
            return True
        if self._state == CircuitState.OPEN:
            if self._last_failure_time is not None:
                elapsed = time.time() - self._last_failure_time
                if elapsed >= self.recovery_timeout:
                    self._transition_to(CircuitState.HALF_OPEN)
                    return True
            return False
        return self._state == CircuitState.HALF_OPEN

async def record_failure(self) -> None:
    async with self._lock:
        self._consecutive_failures += 1
        self._last_failure_time = time.time()
        if self._state == CircuitState.HALF_OPEN:
            self._transition_to(CircuitState.OPEN)
        elif self._state == CircuitState.CLOSED and self._consecutive_failures >= self.failure_threshold:
            self._transition_to(CircuitState.OPEN)
```

### D. Why It Was Implemented This Way
- **Concurrency Safety via `asyncio.Lock`:** Python coroutines in an ASGI event loop yield control during `await`. Without an `asyncio.Lock()`, multiple concurrent coroutines checking `can_execute()` simultaneously when the timer expires could all enter `HALF_OPEN`, flooding the recovering upstream service with probes instead of a controlled canary.
- **Fail-Fast Mechanics in Router:** In [app/services/router.py#L108-L126](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py#L108-L126), when `can_execute()` returns `False`, the router dispatches directly to fallback without awaiting network timeouts.

### E. Interview Follow-Up Questions & Production Answers
- **Q: Is this circuit breaker distributed across multiple server instances?**  
  *Answer:* Currently, this circuit breaker is process-local within each worker's memory. In a large cluster, we would synchronize state across pods using Redis Hash (`HSET circuit:groq state OPEN last_failure 1718000000`) and Pub/Sub for immediate notification.
- **Q: Why consecutive failures instead of a percentage-based error rate?**  
  *Answer:* For external LLM inference, 5 consecutive HTTP 503/429 errors indicate a complete vendor outage or hard quota exhaustion. Consecutive tracking is deterministic and does not suffer from cold-start sample-size inaccuracies.

### F. Simplified Illustrative Implementation
```python
# Simplified Circuit Breaker
import time

class SimpleCircuitBreaker:
    def __init__(self, threshold=5, recovery_timeout=60.0):
        self.state = "CLOSED"
        self.failures = 0
        self.threshold = threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure = 0.0

    def can_execute(self) -> bool:
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            if time.time() - self.last_failure >= self.recovery_timeout:
                self.state = "HALF_OPEN"
                return True
            return False
        return self.state == "HALF_OPEN"

    def record_success(self):
        self.state = "CLOSED"
        self.failures = 0

    def record_failure(self):
        self.failures += 1
        self.last_failure = time.time()
        if self.failures >= self.threshold or self.state == "HALF_OPEN":
            self.state = "OPEN"
```

### G. Modification Exercise
- **Task:** Add a Prometheus counter that increments every time the Circuit Breaker trips to `OPEN`.
- **Files to Change:** [app/core/telemetry.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/telemetry.py), [app/core/circuit_breaker.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py#L150).
- **Approach:** Define `CIRCUIT_BREAKER_TRIPPED_TOTAL = Counter(...)` in telemetry. In `CircuitBreaker._transition_to`, call `if new_state == CircuitState.OPEN: CIRCUIT_BREAKER_TRIPPED_TOTAL.labels(circuit=self.name).inc()`.

---

## Feature 5: Distributed Token-Bucket Rate Limiter with Atomic Redis Lua Scripting

### A. Purpose
- **Problem Solved:** APIs without rate limiting are vulnerable to Denial of Service (DoS), runaway billing from malicious scripts, and noisy neighbors who exhaust upstream LLM quotas.
- **Why Needed in SentinelRAG:** SentinelRAG is a multi-tenant gateway. Each tenant must be guaranteed their allocated throughput without impacting other organizations.
- **What Happens Without It:** A single misconfigured client script making a tight loop of requests burns through the entire company's OpenAI/Groq API key quota within minutes.

### B. Core Concept
The Token Bucket algorithm maintains a bucket with a maximum capacity of tokens. Tokens replenish at a constant rate per second. Each arriving request consumes 1 token. If the bucket is empty, the request is rejected immediately with HTTP 429 Too Many Requests.

---

### C. Actual Implementation in SentinelRAG
- **File:** [app/core/rate_limiter.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py)
- **Classes:** [TokenBucketLimiter](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L82-L168), [TokenBucketRateLimitMiddleware](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L171-L268)
- **Redis Lua Script:** [TOKEN_BUCKET_LUA_SCRIPT](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L17-L69)

```lua
-- Code Excerpt: app/core/rate_limiter.py (Lines 30-68)
local time_arr = redis.call('TIME')
local now = tonumber(time_arr[1]) + (tonumber(time_arr[2]) / 1000000)

local current_data = redis.call('HMGET', key, 'tokens', 'last_updated')
local tokens = tonumber(current_data[1])
local last_updated = tonumber(current_data[2])

if tokens == nil or last_updated == nil then
    tokens = capacity
    last_updated = now
else
    local delta = math.max(0, now - last_updated)
    tokens = math.min(capacity, tokens + (delta * refill_rate))
    last_updated = now
end

if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
else
    allowed = 0
    retry_after = math.ceil((requested - tokens) / refill_rate)
end

redis.call('HSET', key, 'tokens', tokens, 'last_updated', last_updated)
redis.call('EXPIRE', key, math.max(60, math.ceil((capacity / refill_rate) * 2)))
return {allowed, math.floor(tokens), retry_after, capacity}
```

### D. Why It Was Implemented This Way
- **Why Redis Lua instead of Python Logic?** If two requests arrive simultaneously across two Uvicorn workers, both would read `tokens = 1`, decrement to `0`, and write back, permitting 2 requests when only 1 should be allowed (race condition). Executing inside an atomic Redis Lua script guarantees execution on Redis's single-threaded event loop without locks.
- **Why `redis.call('TIME')`?** Uses the Redis server's monotonic clock rather than the application worker's clock, preventing clock-skew synchronization errors across distributed servers.
- **Fail-Open Policy:** If Redis suffers a network timeout or crash ([app/core/rate_limiter.py#L161-L168](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L161-L168)), the limiter catches the exception, logs an alert, and returns `allowed=True`. In production, temporary monitoring degradation should never take down user traffic.

### E. Interview Follow-Up Questions & Production Answers
- **Q: Why use Token Bucket over Leaky Bucket or Fixed Window?**  
  *Answer:* Fixed windows suffer from the 2x burst boundary problem (100 requests at 11:59:59 and 100 at 12:00:01). Token Bucket supports legitimate instantaneous bursts up to capacity while enforcing a strict long-term sustained refill rate.
- **Q: How does the script prevent Redis memory leaks?**  
  *Answer:* The script sets an automatic expiration TTL: `math.max(60, (capacity / refill_rate) * 2)`. Inactive client buckets expire automatically from Redis memory.

### F. Simplified Illustrative Implementation
```python
# Simplified In-Memory Token Bucket for Conceptual Defense
import time

class MemoryTokenBucket:
    def __init__(self, capacity: int = 10, refill_per_sec: float = 1.0):
        self.capacity = capacity
        self.refill = refill_per_sec
        self.tokens = float(capacity)
        self.last_update = time.time()

    def allow(self) -> bool:
        now = time.time()
        delta = now - self.last_update
        self.tokens = min(self.capacity, self.tokens + delta * self.refill)
        self.last_update = now

        if self.tokens >= 1.0:
            self.tokens -= 1.0
            return True
        return False
```

### G. Modification Exercise
- **Task:** Support weighted rate limiting (e.g., chat completions consume 5 tokens, while health checks consume 1 token).
- **Files to Change:** [app/core/rate_limiter.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L215-L220).
- **Approach:** In `TokenBucketRateLimitMiddleware.dispatch`, inspect `request.url.path`. Pass `requested = 5 if "completions" in path else 1` to `self.limiter.check(..., requested=requested)`.

---

## Feature 6: Resilient Multi-Provider LLM Router with Tenacity Full Jitter

### A. Purpose
- **Problem Solved:** Upstream LLMs fail constantly under load with HTTP 429 (Rate Limit) and HTTP 503 (Capacity Exhausted). Retrying immediately without delay makes provider outages worse (Thundering Herd problem).
- **Why Needed in SentinelRAG:** SentinelRAG ensures 99.95% API gateway availability regardless of individual provider status.
- **What Happens Without It:** Any transient blip in Groq or OpenAI results in an immediate HTTP 500 error returned to the end user.

### B. Core Concept
Combines **Exponential Backoff with Full Jitter** and **Automated Multi-Provider Failover**. When an error occurs, the wait time is chosen uniformly at random between 0 and the exponential upper bound:
$$T_{\text{sleep}} \sim \text{Uniform}\left(0, \min(T_{\text{max}}, \text{multiplier} \cdot 2^{\text{attempt}})\right)$$
Full Jitter completely decorrelates retry storms from hundreds of concurrent clients.

---

### C. Actual Implementation in SentinelRAG
- **Files:** [app/services/router.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py), [app/services/resilience.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/resilience.py)
- **Transient Exception Filter:** [is_transient_error(exc)](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/resilience.py#L29-L76). Identifies 429, 503, 502, 504, and socket disconnects while failing fast on permanent client errors (400, 401, 403, 404).

```python
# Code Excerpt: app/services/resilience.py (Lines 102-108)
return AsyncRetrying(
    retry=retry_if_exception(is_transient_error),
    wait=wait_random_exponential(multiplier=multiplier, max=max_wait),
    stop=stop_after_attempt(max_attempts),
    before_sleep=before_sleep_hook,
    reraise=True,
)
```

### D. Why It Was Implemented This Way
- **Separation of Concerns:** `resilience.py` encapsulates pure retry mechanics using Tenacity. `router.py` encapsulates business routing logic (Groq $\to$ OpenAI $\to$ Mistral).
- **Timeout Protection:** Every call passes a hard 4.0-second timeout (`timeout=settings.CIRCUIT_BREAKER_TIMEOUT_SECONDS`) to prevent hung TCP sockets.

### E. Interview Follow-Up Questions & Production Answers
- **Q: Why not retry HTTP 401 Unauthorized?**  
  *Answer:* HTTP 401 indicates an invalid API key. Retrying will never succeed and merely adds latency and wastes CPU cycles. Permanent errors must fail-fast immediately.
- **Q: What is the difference between Equal Jitter and Full Jitter?**  
  *Answer:* Full Jitter chooses $U(0, \text{backoff})$. Equal Jitter preserves half the backoff and jitters the other half: $\frac{\text{backoff}}{2} + U(0, \frac{\text{backoff}}{2})$. AWS architectural whitepapers demonstrated that Full Jitter minimizes client competition and overall server queue depth best.

### F. Simplified Illustrative Implementation
```python
# Simplified Retry with Exponential Backoff and Jitter
import asyncio, random

async def retry_with_jitter(coro_fn, max_attempts=3, base=0.5):
    for attempt in range(max_attempts):
        try:
            return await coro_fn()
        except Exception as e:
            if attempt == max_attempts - 1:
                raise
            max_sleep = base * (2 ** attempt)
            sleep_time = random.uniform(0, max_sleep)
            await asyncio.sleep(sleep_time)
```

### G. Modification Exercise
- **Task:** Add dynamic header-based timeout overrides from clients (`X-Request-Timeout-Seconds`).
- **Files to Change:** [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py), [app/services/router.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py).
- **Approach:** Read request header, cap at 10.0 seconds maximum to prevent abuse, and pass to `router_service.complete(..., timeout=custom_timeout)`.

---

## Feature 7: Real-Time BPE Token Counting & Cumulative Spend Analytics

### A. Purpose
- **Problem Solved:** LLM providers bill asynchronously at the end of the month. Without real-time internal token accounting, organizations experience billing shock or cannot attribute costs across departments (chargebacks).
- **Why Needed in SentinelRAG:** Quantifies the exact return on investment (ROI) of SentinelRAG by calculating the cumulative dollars saved by the semantic cache.
- **What Happens Without It:** Product teams have zero visibility into per-tenant inference costs or cache ROI.

### B. Core Concept
Byte-Pair Encoding (BPE) tokenization splits text into sub-word tokens. SentinelRAG computes exact prompt tokens (accounting for message formatting metadata `<|start|>role...`) and completion tokens, applying a granular pricing table to track spend and savings in atomic Redis counters.

---

### C. Actual Implementation in SentinelRAG
- **File:** [app/services/token_analytics.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/token_analytics.py)
- **Class:** [TokenAnalyticsTracker](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/token_analytics.py#L91-L226)
- **Token Functions:** `count_messages_tokens(messages, model)` and `count_text_tokens(text, model)`.
- **Pricing Table:** [MODEL_PRICING](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/token_analytics.py#L18-L32) storing prompt and completion costs per 1,000,000 tokens for Groq, OpenAI, and Mistral.

```python
# Code Excerpt: app/services/token_analytics.py (Lines 70-80, 120-128)
# BPE message markup accounting
num_tokens = 0
for message in messages:
    num_tokens += 3  # <|start|>{role}\n{content}<|end|>\n
    for key, value in message.items():
        num_tokens += len(tokenizer.encode(str(value)))
num_tokens += 3  # <|start|>assistant<|message|>

# Atomic Redis pipeline increment
pipe = self.redis.pipeline()
pipe.incrby(f"analytics:tenant:{tenant_id}:prompt_tokens", prompt_tokens)
pipe.incrbyfloat(f"analytics:tenant:{tenant_id}:dollars_spent", cost)
await pipe.execute()
```

### D. Why It Was Implemented This Way
- **Why tiktoken over rough word approximations?** Estimating tokens as `len(text) // 4` has errors up to $\pm 30\%$ on code snippets, JSON, and non-English text. `tiktoken` gives exact BPE tokens matching OpenAI and Groq tokenizers.
- **Why Atomic Pipeline Counters?** Storing spend metrics via Redis `INCRBY` and `INCRBYFLOAT` allows high-concurrency increments across stateless workers without database write locks.

### E. Interview Follow-Up Questions & Production Answers
- **Q: What happens if `tiktoken` does not recognize a model identifier like `groq/llama-3.3-70b`?**  
  *Answer:* [get_tokenizer_for_model](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/token_analytics.py#L50-L58) traps exceptions and falls back gracefully to `cl100k_base` (the standard BPE tokenizer for modern GPT and Llama architectures).
- **Q: How does the system calculate "Dollars Saved"?**  
  *Answer:* When a semantic cache hit occurs, [record_cache_hit_savings](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/token_analytics.py#L134) calculates what that request *would* have cost using baseline OpenAI `gpt-4o-mini` rates. It increments `analytics:tenant:{tenant_id}:dollars_saved` in Redis and increments the `cost_saved_usd_total` Prometheus counter.

### F. Simplified Illustrative Implementation
```python
# Simplified Token Cost Tracker
def calculate_cost(prompt_tokens: int, completion_tokens: int, model: str = "gpt-4o-mini") -> float:
    rates = {"prompt": 0.15, "completion": 0.60}  # $ per 1M tokens
    cost = (prompt_tokens / 1e6) * rates["prompt"] + (completion_tokens / 1e6) * rates["completion"]
    return round(cost, 6)
```

### G. Modification Exercise
- **Task:** Support token budget enforcement (e.g., reject requests if a tenant exceeds \$500/month).
- **Files to Change:** [app/services/token_analytics.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/token_analytics.py), [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py).
- **Approach:** Add method `async def check_budget(tenant_id, max_dollars) -> bool`. Read `analytics:tenant:{tenant_id}:dollars_spent`. If exceeded, raise a custom `ProblemDetailException(status=402, title="Payment Required", detail="Tenant token budget exhausted")`.
