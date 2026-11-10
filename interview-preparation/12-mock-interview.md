# 12. Simulated 4-Round Mock Interview

Practice these four timed 45-minute mock rounds **out loud**—without looking at your notes during each answer. Each prompt lists the interviewer's **Hidden Probe Intent**, **Trap / Weak Answers to Avoid**, and a **Model Answer Outline** verified against [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md) and [kiranteja-ai-engineer.pdf](file:///c:/Users/kiran/Desktop/notes/resume/kiranteja-ai-engineer.pdf).

> **How to Run This Mock:**
> 1. Set a 45-minute timer per round. Draw diagrams on paper or a blank whiteboard before speaking.
> 2. Speak in the **20–40 second Short Answer $\to$ Structured Deep Dive** cadence so you do not monologue past 2 minutes before letting the interviewer steer.
> 3. Respect the **Honesty Rule**: present **SentinelRAG** accurately as an in-progress independent engineering/architecture project (not a live employer production deployment), and verify any **KloudWorx** metrics against what you personally observed and delivered at KloudWorx.

---

## Round 1: SentinelRAG Architecture & Whiteboard Design (45 Mins)

### Prompt 1.1 — System Walkthrough & Motivation
> **Interviewer:** *"Walk me through SentinelRAG on the whiteboard. What problem were you trying to solve, and trace a single `POST /api/v1/chat/completions` request end-to-end."*

- **Hidden Probe Intent:** Tests whether you understand system boundaries, component ordering, and the *exact* codebase behavior rather than a generic textbook RAG diagram.
- **Trap / Weak Answers:**
  - *Trap:* "We ran this in production for 500 enterprise customers and cut LLM costs by 54.2%." *(Fictional claim from an unverified note; SentinelRAG is an in-progress portfolio project tested with mocked backends.)*
  - *Trap:* "The rate limiter runs first, then FastAPI generates embeddings, queries RediSearch `FT.SEARCH`, and runs dense + sparse in parallel with `asyncio.gather`." *(Four factual inaccuracies in one sentence!)*
- **Model Answer:**
  1. **Framing:** "I built SentinelRAG as an independent engineering project to solve three common bottlenecks in RAG backends: redundant LLM calls/spend, poor keyword precision in dense-only vector search, and brittle external LLM provider dependencies."
  2. **Trace (Draw 5 boxes as you speak):**
     - **Edge & Middleware:** Nginx forwards to FastAPI. Because Starlette applies middleware LIFO, `@app.middleware("http")` (`correlation_id_and_timing_middleware`) executes **first** to bind `X-Request-ID` into `contextvars` and record Prometheus latency, followed by `CORSMiddleware`, then `TokenBucketRateLimitMiddleware`.
     - **Rate Limiter:** Runs an atomic Redis Lua token bucket (`capacity=100`, `refill=10/s`) keyed by `X-Tenant-ID`, using Redis `TIME` and **failing open** on `RedisError`.
     - **Two-Tier Semantic Cache:** `RedisSemanticCache.evaluate()` checks exact normalized query hash via **SHA-256** (`HGETALL cache:{tenant_id}:{query_hash}`). On miss, if the caller supplied `query_vector`, it fetches `SMEMBERS cache:{tenant_id}:keys` + pipelined `HGETALL` and scans Base64-packed float vectors in Python, returning a hit if cosine distance $\le 0.08$.
     - **Single-Store Hybrid Retrieval:** On cache miss, `SearchRepository` **sequentially** awaits two SQL queries on PostgreSQL 16 (`pgvector` HNSW cosine `<=>` and GIN persisted `tsvector` `ts_rank_cd`), fetching `top_k_chunks * 2` from each, then merges via RRF (`k=60`).
     - **Resilient LLM Router & Telemetry:** `ResilientLLMRouter` routes through Groq (`max_attempts=3`, 4s timeout, guarded by a 5-failure/60s `CircuitBreaker`), failing over to OpenAI (`max_attempts=2`) and Mistral (`1` call), tracks token counts via `tiktoken`, writes the cache inline, and returns JSON with `X-Request-ID`.

---

### Prompt 1.2 — Deep Probe on Data Modeling & Multi-Tenancy
> **Interviewer:** *"Wait—how do you guarantee Tenant A never sees Tenant B's cached answers or retrieved document chunks?"*

- **Hidden Probe Intent:** Checks defense-in-depth vs. real gaps in the current implementation.
- **Trap / Weak Answers:**
  - *Trap:* "FastAPI validates JWT scopes in auth middleware and applies PostgreSQL Row-Level Security (RLS)." *(Neither auth middleware nor RLS policies are implemented in the current code!)*
- **Model Answer:**
  - **Currently Implemented Isolation:** Every cache key (`cache:{tenant_id}:{query_hash}` and `cache:{tenant_id}:keys`), rate-limit key (`ratelimit:{tenant_id}`), and SQL query (`WHERE tenant_id = :tenant_id` against an indexed `DocumentChunk.tenant_id` column) explicitly namespaces by `X-Tenant-ID` / `tenant_id`.
  - **Honest Security Gap & Production Next Step:** "Right now `X-Tenant-ID` defaults cleanly for gateway prototyping rather than enforcing cryptographic auth. Before exposing this multi-tenant gateway externally, I would add JWT/API-key authentication middleware that derives `tenant_id` from a signed claim rather than a raw client header, and pair application filters with PostgreSQL Row-Level Security (`FORCE ROW LEVEL SECURITY` + `SET LOCAL app.tenant_id`) as defense-in-depth."

---

### Prompt 1.3 — Probe on Semantic Cache Internals
> **Interviewer:** *"You mentioned scanning `SMEMBERS` + `HGETALL` in Python. Walk me through why you didn't just store JSON arrays in Redis, and what happens when expired keys accumulate in the tracking set."*

- **Hidden Probe Intent:** Tests low-level data encoding (`struct.pack` + Base64 + `decode_responses=True`) and set-TTL drift cleanup.
- **Model Answer:**
  - **Binary Packing + Base64:** JSON text for `1536` floats wastes $\sim 18\text{ KB}$ per vector and adds JSON parsing CPU. Packing `struct.pack(f"{len(vector)}f", *vector)` produces exact `6,144` bytes (a ~3x storage reduction even after Base64 expansion to `8,192` chars). Base64 is required specifically because `RedisManager` initializes the shared client with `decode_responses=True` so Lua scripts and string payloads stay UTF-8 clean.
  - **Ghost Key Cleanup:** Redis expires individual hash keys (`TTL=86400s`), not members inside `cache:{tenant_id}:keys`. During `evaluate()`, any hash returning empty (`not hdata`) triggers `await redis.srem(index_key, ghost_key)` lazily, keeping the index bounded without background `SCAN` jobs. *(Bonus point: mention RediSearch `FT.SEARCH` HNSW as the upgrade path once a tenant exceeds ~5,000 keys.)*

**Review links for Round 1:** [02-project-architecture.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/02-project-architecture.md), [06/02-rate-limiter-and-semantic-cache.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md).

---

## Round 2: Python, AsyncIO, SQLAlchemy Internals & Live Debugging (45 Mins)

### Prompt 2.1 — AsyncIO Event Loop & FastAPI Blocking
> **Interviewer:** *"Suppose a new engineer accidentally switches one of our helper calls inside an `async def` chat endpoint to synchronous `requests.post(...)` or runs a slow 200ms `tiktoken` call over a 500KB document in the event loop. What breaks, and how does FastAPI handle `def` vs `async def`?"*

- **Hidden Probe Intent:** Tests event-loop single-thread semantics, starvation of health checks/other tenants, and AnyIO threadpool behavior.
- **Model Answer:**
  - An `async def` route executes directly on Uvicorn's single-threaded event loop. A blocking network call (`requests.post`) or CPU-bound loop starves *every* in-flight coroutine—meaning `/healthz` fails Kubernetes/Docker liveness probes and unrelated tenants stall even though `asyncpg` and Redis are idle.
  - FastAPI avoids this automatically for plain `def` endpoints by dispatching them to AnyIO's worker threadpool (`~40` threads by default). Inside an `async def` route, blocking I/O must be replaced with native async clients (`httpx`/`litellm.acompletion`), and heavy CPU work (`tiktoken` on huge blobs or big NumPy scans) offloaded via `asyncio.to_thread()` or a process pool.

---

### Prompt 2.2 — SQLAlchemy `AsyncSession` Pitfalls
> **Interviewer:** *"Why did you set `expire_on_commit=False` in `AsyncSessionLocal`, and why did `SearchRepository` NOT run `search_dense` and `search_sparse` concurrently with `asyncio.gather()`?"*

- **Hidden Probe Intent:** Differentiates engineers who actually wired `SQLAlchemy 2.0 + asyncpg` from people copying boilerplate.
- **Model Answer:**
  1. **`expire_on_commit=False`:** By default, SQLAlchemy expires ORM attributes on `commit()`. In async mode, later accessing an expired attribute triggers an **implicit lazy SQL load**, which raises `MissingGreenlet` because property access isn't `await`able. Setting `expire_on_commit=False` lets committed attributes remain readable in memory cleanly.
  2. **No `asyncio.gather` on one `AsyncSession`:** SQLAlchemy's `AsyncSession` checks out a single `asyncpg` wire connection per transaction/unit-of-work. `asyncpg` rejects overlapping protocol commands on one connection with `asyncpg.exceptions.InterfaceError: cannot perform operation: another operation is in progress`. So `SearchRepository` executes `await self.search_dense(...)` then `await self.search_sparse(...)` sequentially (or else would need two separate sessions from `async_sessionmaker`).

---

### Prompt 2.3 — Live Debugging & Code Review Drill
> **Interviewer:** *"Look at these two snippets from SentinelRAG. Spot the real concurrency bug in the first and the adversarial input bug in the second, then tell me how you'd fix both."*

```python
# Snippet A: CircuitBreaker.__init__ & call() (simplified)
async def call(self, func, *args, **kwargs):
    async with self._lock:
        if self._state == CircuitState.OPEN:
            if time.monotonic() - self._last_failure_time >= self._recovery_time:
                self._state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpenError(self.provider)
    try:
        result = await asyncio.wait_for(func(*args, **kwargs), timeout=self._timeout)
    except Exception as exc:
        await self._on_failure()
        raise
    await self._on_success()
    return result

# Snippet B: TokenAnalytics.count_tokens()
def count_tokens(self, text: str) -> int:
    if not text:
        return 0
    return len(self._encoder.encode(text))
```

- **Hidden Probe Intent:** Can you debug your *own* code critically?
- **Model Answer:**
  - **Bug in Snippet A (`HALF_OPEN` Probe Race):** `_lock` is released *before* `await asyncio.wait_for(func(...))`. When `recovery_time` (`60s`) elapses, 50 concurrent coroutines can all enter `_lock`, transition/observe `HALF_OPEN`, exit `_lock`, and fire 50 simultaneous trial requests into a recovering provider instead of 1 canary probe. **Fix:** Track `_probe_in_flight: bool = False`. Inside `_lock`, if state is `HALF_OPEN`: if `_probe_in_flight` is already `True`, raise `CircuitBreakerOpenError`; otherwise set `_probe_in_flight = True` and reset it in a `try/finally` inside `_on_success` / `_on_failure` (or `finally`).
  - **Bug in Snippet B (`tiktoken` Special-Token Injection):** `self._encoder.encode(text)` defaults to throwing `ValueError` if untrusted user text contains special BPE delimiters like `<|endoftext|>`. **Fix:** Pass `disallowed_special=()` (treating special tokens as plain text instead of throwing a 500 error).

**Review links for Round 2:** [05/01-fastapi-asgi](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/01-fastapi-asgi-and-asyncio.md), [05/02-postgresql](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/02-postgresql-pgvector-and-fts.md), [07/E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md).

---

## Round 3: Resilience, Adversarial Trade-Offs & 100x Scaling (45 Mins)

### Prompt 3.1 — Single-Store Postgres vs. Pinecone + Elasticsearch
> **Interviewer:** *"Why didn't you use Pinecone for vectors and Elasticsearch for keyword search? Isn't `pgvector` in PostgreSQL a toy at scale?"*

- **Hidden Probe Intent:** Tests systems pragmatism vs. resume-driven architecture.
- **Model Answer:**
  - **Short Answer:** For SentinelRAG's target scope, a single PostgreSQL store with `pgvector` HNSW + GIN `tsvector` eliminates dual-write drift, keeps tenant metadata filtering ACID-consistent inside one `WHERE` clause, and halves operational overhead (1 backup/migration story vs. 3).
  - **Know the Exact Breakpoint:** `pgvector` HNSW stores its graph in memory/shared buffers. Once vector volume crosses ~10M+ chunks on a single node (causing HNSW RAM pressure to fight PostgreSQL `shared_buffers`), or write/update QPS saturates single-primary WAL, or recall/latency needs distributed sharded ANN + native BM25 with custom analyzers, I would extract search to Qdrant/Pinecone + OpenSearch and maintain consistency via the **Transactional Outbox Pattern** (CDC/worker from Postgres).

---

### Prompt 3.2 — Redis Failure During a Traffic Spike
> **Interviewer:** *"You configured `TokenBucketRateLimitMiddleware` to fail open when Redis raises `RedisError`. What happens if Redis goes down during a brute-force or DDoS spike?"*

- **Hidden Probe Intent:** Sees whether you recognize the trade-off behind every failure policy.
- **Model Answer:**
  - Failing open prioritizes **availability** over strict throttling: a brief Redis blip doesn't break paying tenants when the gateway can't check counters.
  - The risk under an active flood is exposing PostgreSQL and LLM budgets. **Mitigation:** Pair Redis fail-open with two independent layers: (1) edge `limit_req` / WAF rules in Nginx (pure C, zero Redis dependency) as a coarse IP ceiling, and (2) an in-process emergency fallback limiter (smaller per-worker token bucket activated only while Redis circuit/health flags degraded), plus alerting on `cache_operations_total` / Redis error spikes.

---

### Prompt 3.3 — 100x Scale Evolution (Stateless $\to$ Multi-Worker / Multi-Node)
> **Interviewer:** *"Your Dockerfile currently runs `--workers 1`. Walk me through the exact three things that break the moment we bump to `--workers 4` across 3 ECS/Kubernetes pods."*

- **Hidden Probe Intent:** Tests distributed state awareness grounded in your actual code.
- **Model Answer:**
  1. **`CircuitBreaker` Split-Brain:** Currently held in process memory (`self._breakers` dict). One worker opens Groq's breaker while 3 others keep hitting Groq, and pod restarts wipe state. **Fix:** Move state transitions to an atomic Redis Lua hash (`INCR` failure count, `SET` open deadline) or an Envoy/API-gateway circuit breaker.
  2. **Prometheus `/metrics` Fragmentation:** Default `prometheus_client` reads process memory, so scraping load-balanced pods/workers gives chaotic jumps. **Fix:** For multi-worker Python, enable `PROMETHEUS_MULTIPROC_DIR`; better yet in Kubernetes, keep **1 Uvicorn worker per container** (one process = one CPU limit) and let Prometheus scrape every pod IP cleanly.
  3. **DB Connection Multiplication:** `pool_size=20 + max_overflow=10` means up to `30` connections *per worker*. 12 pods/workers $\times 30 = 360$ connections, exhausting Postgres RAM. **Fix:** Put **PgBouncer** in `transaction` pooling mode in front of Postgres, reduce app pool sizes, and pass `connect_args={"statement_cache_size": 0}` to `asyncpg`.

**Review links for Round 3:** [08-scenario-based-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/08-scenario-based-questions.md), [09-design-decisions-and-tradeoffs.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md), [10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md).

---

## Round 4: Resume Defense, KloudWorx & Behavioral STAR (45 Mins)

### Prompt 4.1 — Opening Pitch & M.Sc. Chemistry Pivot
> **Interviewer:** *"Walk me through your background. I notice an M.Sc. in Chemistry before moving into software and AI engineering—tell me about that journey and your ~3 years at KloudWorx."*

- **Trap / Weak Answers:**
  - *Trap:* Sounding defensive about Chemistry or spending 2 minutes talking about 2018 coursework before reaching KloudWorx production systems.
- **Model Answer Structure (~90 seconds):**
  1. **Present Impact First (KloudWorx Apr 2023–Jan 2026):** "For nearly 3 years at KloudWorx Technologies, I worked as a backend developer building enterprise HRTech, digital onboarding, e-signature, and government-facing platforms—shipping 100+ REST APIs across Python/FastAPI and PHP Slim."
  2. **Core Engineering Proof:** Mention async decoupling with AWS SQS & idempotent workers, direct-to-S3 pre-signed URLs, MySQL indexing/stored procedures + Redis cache-aside optimizations, on-prem Elasticsearch nested mapping, and Stripe HMAC-signed async webhook processing.
  3. **Bridge to AI Systems:** "Outside work I've been applying those backend reliability patterns to LLM infrastructure through **SentinelRAG**—solving semantic caching, pgvector hybrid retrieval, and multi-provider circuit-broken routing."
  4. **Chemistry $\to$ Engineering Pivot (15s):** "My M.Sc. in Chemistry trained me in hypothesis-driven debugging, isolating variables, and rigorous reproducibility—which translated directly from TA mentoring at CCBP Intensive into diagnosing production SQL bottlenecks and distributed race conditions."

---

### Prompt 4.2 — KloudWorx Architecture Deep Probe (SQS + S3 + Stripe + MySQL)
> **Interviewer:** *"Pick one production bottleneck you fixed at KloudWorx—say, duplicate webhook/queue execution or slow document uploads—and tell me the exact mechanics."*

- **Hidden Probe Intent:** Ensures your KloudWorx resume bullets reflect real engineering mechanics, not buzzwords.
- **Model Answer Checklist (Choose the mechanic cleanly; state only metrics you personally stand behind):**
  - **If choosing Stripe / SQS Idempotency:**
    - Signature verification (`Stripe-Signature` HMAC-SHA256 + timestamp delta $< 300\text{s}$ replay window via constant-time compare).
    - **Dual Idempotency:** Return HTTP `200` fast ($< 5\text{s}$ timeout window) while persisting/checking `event.id` (`processed_webhooks` unique constraint) and pushing async jobs to **SQS** with a `UUIDv4` idempotency key + worker-side Redis `SET key 1 NX EX 300` lock + DLQ redrive (`maxReceiveCount=3`) for poison messages.
  - **If choosing S3 Pre-Signed Uploads:**
    - Replaced API-proxied file uploads (which held worker memory/connections hostage) with backend-generated S3 pre-signed URLs (`ExpiresIn=900`, `AES256`) so mobile/web clients upload directly to S3.
    - Verified file authenticity asynchronously via **magic bytes** (`Range='bytes=0-7'`: `%PDF-`, `\x89PNG`) rather than trusting spoofable `Content-Type` headers.

---

### Prompt 4.3 — Behavioral STAR & Ownership Boundaries
> **Interviewer:** *"Tell me about a time you pushed back on a technical decision or diagnosed a difficult production slowdown."*

- **Hidden Probe Intent:** Evaluates conflict resolution (data over ego), root-cause methodology, and whether you own follow-up prevention.
- **Execution Checklist (STAR):**
  - **S (Situation):** Frame concrete production context from KloudWorx (e.g., frontend proposing Base64-encoded PDF payloads inside JSON, OR slow reporting queries / CPU spike from missing composite indexes / lock contention). *(Verify exact numbers against your real KloudWorx experience before stating them!)*
  - **T (Task):** Protect API worker availability/latency without blocking the feature deadline.
  - **A (Action):** Measured payload/query impact objectively (`EXPLAIN ANALYZE`, memory/timeout math), proposed the safer alternative (S3 pre-signed URL hook or ESR composite indexing + zero-downtime schema plan), and shipped a concrete helper/template so teammates adopted it effortlessly.
  - **R (Result):** Summarize the observable improvement and **systemic prevention** added afterward (shared FastAPI starter template, Pytest rollback fixtures, query review habit).

**Review links for Round 4:** [14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md).

---

## Post-Mock Self-Scoring Rubric (Grade Honestly Out of 20)

Score **1–5** per round immediately after finishing:

| Dimension | 1–2 (Needs Drill) | 3–4 (Solid Mid-Level) | 5 (Strong Senior Signal) |
| :--- | :--- | :--- | :--- |
| **Factual Accuracy** | Mixes up MD5/SHA-256, RediSearch/Python scan, or invents production metrics | Core architecture accurate; small constant slips | 100% consistent with code; cleanly separates implemented vs. planned |
| **Mechanism Depth** | "Redis does rate limiting" without knowing Lua/atomicity | Explains Lua atomicity and `TIME` | Explains Lua + `EVALSHA`, float drift, fail-open risk, and `SMEMBERS` ghost `SREM` |
| **Trade-Off Awareness** | Claims chosen stack has zero downsides | Mentions 1 basic alternative | Quantifies RAM/latency/ops trade-offs and states exact migration triggers |
| **Conciseness & Clarity** | Wanders for 4+ minutes without answering the prompt | Clear explanation in ~90s | Crisp 20–40s hook, then invites interviewer to choose depth |

