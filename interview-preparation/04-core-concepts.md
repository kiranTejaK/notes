# 04. Core Concepts From First Principles

This document distills the six foundational computer science, distributed systems, and information retrieval concepts underlying SentinelRAG. For implementation line-by-line details, follow the links to [05-technology-deep-dives/](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/01-fastapi-asgi-and-asyncio.md) and [06-implementation-deep-dives/](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/01-app-lifecycle-and-middleware.md).

---

## 1. Cooperative Async I/O (`asyncio` Event Loop) & `contextvars`

- **Fundamental Idea:** Traditional WSGI servers dedicate an OS thread per HTTP request. When a thread blocks waiting 1.5s for an LLM API or DB query, it sits idle while consuming ~2–8MB of stack RAM and forcing kernel context switches. ASGI (`async def app(scope, receive, send)`) runs a single-threaded event loop backed by OS I/O multiplexing (`epoll` on Linux). Whenever a coroutine reaches `await` on non-blocking network I/O (`asyncpg`, `redis.asyncio`, `httpx`), it suspends and lets the loop run other ready coroutines.
- **The Blocking Trap:** Because there is only one thread per worker event loop, calling synchronous blocking I/O (`requests.get()`, `time.sleep()`, `psycopg2`) or heavy CPU loops inside `async def` freezes the *entire worker* for all concurrent clients—even `/healthz`.
- **Context Propagation (`contextvars`):** `threading.local()` fails in async Python because thousands of coroutines share one OS thread and overwrite each other across `await` yields. `contextvars.ContextVar` attaches state to the logical `asyncio.Task` context (`structlog.contextvars`), and calling `clear_contextvars()` at request ingress prevents stale metadata leaks.

---

## 2. Distributed Token Bucket Rate Limiting (Atomic Redis Lua)

- **Fundamental Idea:** A Token Bucket has a maximum `capacity` (burst allowance, `100` in `config.py`) and refills continuously at `refill_rate` (`10/s`). Each request consumes `requested` tokens (`1`). Rather than running a background timer for every tenant, replenishment is computed **lazily on demand**:
  $$\text{tokens}_{\text{now}} = \min\left(\text{capacity},\; \text{tokens}_{\text{prev}} + \max(0, t_{\text{now}} - t_{\text{prev}}) \times \text{refill\_rate}\right)$$
- **Why Fixed Window & Multi-Step Redis Fail:**
  - *Fixed Window (`INCR` + `EXPIRE`)* allows $2\times$ bursts across window edges (100 req at `11:59:59` + 100 req at `12:00:01`).
  - *Application-side `GET` $\to$ check $\to$ `SET`* causes race conditions across workers/pods.
  - *Redis `MULTI/EXEC`* queues commands blindly and cannot branch conditionally on values read inside the same transaction.
- **Why Redis Lua + `redis.call('TIME')`:** Redis executes Lua scripts atomically inside its single-threaded C event loop—zero interleaving and one network RTT (`EVALSHA`). Reading `redis.call('TIME')` inside Lua uses Redis's single clock instead of worker clocks, eliminating NTP drift across containers.
- **Memory Safety:** Inactive keys expire automatically (`EXPIRE key max(60, ceil(capacity / refill_rate * 2))`).

---

## 3. Vector Embeddings, Cosine Distance & Semantic Gating

- **Fundamental Idea:** Text embeddings map queries into $\mathbb{R}^{1536}$ (e.g., `text-embedding-3-small`) so paraphrases lie close in angle.
- **Distance Metric:**
  $$\text{Cosine Similarity}(\mathbf{u}, \mathbf{v}) = \frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\|_2 \|\mathbf{v}\|_2} \in [-1, 1], \qquad \text{Cosine Distance} = 1 - \text{Cosine Similarity} \in [0, 2]$$
  Why angle instead of Euclidean distance ($L_2$)? Cosine ignores vector magnitude and focuses purely on directional semantic alignment (though for unit-normalized embeddings $\|u\|=\|v\|=1$, Cosine, $L_2^2$, and negative Inner Product `<#>` order identically).
- **Two-Tier Lookup & Threshold (`<= 0.08`):**
  1. **Tier 1:** Exact normalized `SHA-256` hash check ($O(1)$, zero math).
  2. **Tier 2:** Cosine distance scan across tenant cached vectors packed as base64 IEEE 754 `float32` (`6,144` bytes binary $\to$ `8,192` chars base64 vs. ~28–35 KB JSON).
  - **Why `<= 0.08` (similarity $\ge 0.92$)?** Too loose (e.g., `0.15–0.20`) triggers false-positive collisions between related but distinct intents (*"Plan A deductible"* vs *"Plan B deductible"*); too tight (`0.02`) misses genuine paraphrases.

---

## 4. Single-Store Hybrid Search: HNSW + `tsvector` GIN

- **Why Hybrid?**
  - *Dense Vector Search* captures synonyms/concepts (*"car"* $\leftrightarrow$ *"automobile"*) but fails on exact SKUs, error codes (`ERR-9021`), or CVEs (`CVE-2024-38077`) because BPE splits rare alphanumeric tokens.
  - *Sparse Lexical Search* matches exact stemmed lexemes instantly via inverted posting lists (`GIN`) but returns 0 hits when phrasing differs (*"recycle Celery daemon"* vs *"restart worker"*).
- **Why Single-Store PostgreSQL 16?** Storing `chunk_text`, `JSONB` metadata, `Vector(1536)`, and `TSVECTOR` (`GENERATED ALWAYS AS (to_tsvector('english', chunk_text)) STORED`) in one row guarantees ACID updates/deletes without dual-write sync pipelines (Kafka/CDC) to Pinecone/Elasticsearch.
- **Index Mechanics:**
  - **HNSW (`m=16, ef_construction=64, vector_cosine_ops`):** Multi-layer skip-list proximity graph giving $O(\log N)$ approximate search with no k-means training phase (unlike IVFFlat, which degrades on dynamic inserts and cannot train on empty tables).
  - **GIN + `ts_rank_cd`:** Inverted index over lexeme positions; `ts_rank_cd` scores **cover density** (rewarding query terms appearing close together rather than raw word counts).
  - **Operator Matching Trap:** Querying `<->` (L2) against a `vector_cosine_ops` index forces Postgres to abandon HNSW and execute a full `Seq Scan`.

---

## 5. Reciprocal Rank Fusion (RRF, $k=60$)

- **The Problem:** Dense cosine distance ($[0, 2]$, smaller = better) and `ts_rank_cd` ($[0, \infty)$, larger = better) live on incompatible scales and distributions. Min-max scaling is brittle under outliers (one weak query where top sparse score is `0.02` scales to `1.0`).
- **The Formula (Cormack, Clarke & Büttcher, 2009):** Discard raw scores and sum ordinal 1-based ranks:
  $$\text{RRF\_Score}(d) = \sum_{m \in \{\text{dense}, \text{sparse}\}} \frac{1}{k + r_m(d)} \qquad (k = 60)$$
- **Why $k=60$?** Without dampening ($k=0$ or $k=1$), Rank 1 dominates ($1/2 = 0.50$ vs Rank 2 $1/3 = 0.33$). With $k=60$, Rank 1 gets $1/61 \approx 0.01639$ and Rank 2 gets $1/62 \approx 0.01613$ (only ~1.6% gap), so a chunk appearing near the top of **both** lists ($1/62 + 1/62 \approx 0.03226$) easily outranks a chunk appearing #1 in only one list ($0.01639$).

---

## 6. Circuit Breaker FSM & Exponential Backoff with Full Jitter

- **Why Naive Retries Fail:** During an upstream LLM brownout, hanging requests hold open sockets and coroutines until timeouts expire, while fixed-interval retries synchronize into **Thundering Herd** retry waves.
- **Tenacity Full Jitter (Client-Side Transient Recovery):**
  $$T_{\text{sleep}} \sim \text{Uniform}\left(0,\; \min(T_{\text{max}},\; \text{multiplier} \cdot 2^{\text{attempt}})\right)$$
  Randomizing over $[0, \text{cap}]$ spreads retries smoothly across time and prevents retrying non-transient client errors (`400`, `401`, `403`, `404`).
- **Michael Nygard Circuit Breaker (System-Level Protection):**
  - **`CLOSED`:** Traffic hits primary (Groq). Consecutive failures increment; success resets counter to `0`. At `5` failures $\to$ `OPEN`.
  - **`OPEN`:** All requests **fail fast in 0ms** directly to fallback (OpenAI) for `60s`, preventing socket/worker exhaustion and giving Groq time to recover.
  - **`HALF_OPEN`:** After `60s`, allows a canary probe. Success $\to$ `CLOSED`; failure $\to$ immediately back to `OPEN` for another `60s`.
  - **Concurrency Guard:** Protected by `asyncio.Lock` (never `threading.Lock`, which freezes the event loop) so concurrent coroutines don't race past the `HALF_OPEN` transition.

---
**Next:** Explore [05-technology-deep-dives/](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/01-fastapi-asgi-and-asyncio.md) for deep internals and configuration trade-offs.

