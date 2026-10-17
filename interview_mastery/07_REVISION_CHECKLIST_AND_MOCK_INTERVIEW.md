# SentinelRAG: Revision Checklist, Mock Interviews & Self-Assessment Rubric

> **Target Audience:** Python Backend Developers (2–4 Years Experience) conducting final preparation before technical screens and onsite interviews.  
> **Workspace Root:** [c:/Users/kiran/Desktop/sentinel-rag](file:///c:/Users/kiran/Desktop/sentinel-rag)

---

## 1. Comprehensive Interview-Readiness Checklists

Use these checkboxes to verify your mastery of the SentinelRAG codebase and backend engineering concepts before scheduling interviews.

### A. Project Introduction & Architecture
- [ ] Can deliver the 2-minute elevator pitch smoothly without stammering or using filler words.
- [ ] Can deliver the 5-minute technical deep dive covering the 4 pillars (ASGI, Semantic Cache, Hybrid Retrieval, Circuit Breaker).
- [ ] Can explain the exact problem space: cold LLM latency, token cost runaway, vendor rate limits, and lexical search mismatch.
- [ ] Can clearly articulate what is implemented in code versus what is a planned extension.

### B. Request Flows & Code Navigation
- [ ] Can trace `POST /api/v1/chat/completions` through the 4 stages without looking at the documentation.
- [ ] Can open [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py) and locate where prompt augmentation occurs ([lines 200–220](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py#L200-L220)).
- [ ] Can explain why `SearchRepository` executes dense and sparse searches sequentially instead of using `asyncio.gather` on the same session ([app/repositories/search.py#L140](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py#L140)).
- [ ] Can locate where the rate limiter middleware is registered in [app/main.py#L71-L76](file:///c:/Users/kiran/Desktop/sentinel-rag/app/main.py#L71-L76).

### C. Core Feature Implementation Details
- [ ] Can write the Reciprocal Rank Fusion formula from memory: $\sum \frac{1}{k + r_m(d)}$.
- [ ] Can explain why $k=60$ is used in RRF and how it dampens rank disparities.
- [ ] Can explain how the Redis Semantic Cache packs IEEE 754 float vectors using Python's `struct.pack` ([app/services/cache.py#L40-L53](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py#L40-L53)).
- [ ] Can explain the 3 states of the Circuit Breaker (`CLOSED`, `OPEN`, `HALF_OPEN`) and the exact transition triggers ([app/core/circuit_breaker.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py)).
- [ ] Can explain how the atomic Redis Lua script calculates token refills based on `redis.call('TIME')` ([app/core/rate_limiter.py#L30](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py#L30)).

### D. Technology Choices & Trade-Offs
- [ ] Can defend PostgreSQL 16 + `pgvector` vs. dedicated vector databases (Pinecone/Qdrant) using the Dual-System Problem argument.
- [ ] Can defend HNSW over IVFFlat (no pre-training step, $>98\%$ recall, incremental indexing).
- [ ] Can defend RRF over linear score combination (eliminates score normalization skew).
- [ ] Can defend custom LiteLLM + Tenacity router over LangChain / LlamaIndex (debuggability, minimal overhead, deterministic circuit breaking).
- [ ] Can defend FastAPI (ASGI) over Flask/Django (WSGI) for high-concurrency I/O-bound LLM gateways.

### E. Python & Backend Fundamentals
- [ ] Can explain the single-threaded cooperative multitasking event loop in `asyncio`.
- [ ] Can explain why calling a synchronous blocking function inside an `async def` route freezes the entire Uvicorn worker.
- [ ] Can explain why `structlog` uses `contextvars` instead of `threading.local` for request correlation IDs.
- [ ] Can explain how FastAPI's `Depends` with `yield` guarantees rollback and connection cleanup on exceptions.
- [ ] Can explain `expire_on_commit=False` and why lazy-loading fails in async SQLAlchemy.

### F. Coding & Live Debugging
- [ ] Can implement an in-memory Cache-Aside helper with TTL and LRU eviction in live coding.
- [ ] Can implement a basic 3-state Circuit Breaker using `asyncio.Lock` in live coding.
- [ ] Can implement Exponential Backoff with Full Jitter from scratch.
- [ ] Can implement a Token Bucket class with monotonic time replenishment.
- [ ] Can explain how to diagnose an `asyncpg.InterfaceError` and a connection pool exhaustion timeout.

---

## 2. Simulated Mock Interview Rounds

Practice speaking these responses aloud or with a study partner.

---

### Mock Round 1: Architecture & Technical Project Defense (45 Minutes)

#### Question 1: "Tell me about a challenging backend system you built recently."
- **What the Interviewer is Evaluating:** Can the candidate introduce a non-trivial project concisely, establish the problem space, and articulate architectural choices?
- **Strong Candidate Response:** Deliver the 2-minute elevator pitch from [00_PROJECT_MASTER_GUIDE.md](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/00_PROJECT_MASTER_GUIDE.md#L162-L180). Frame SentinelRAG as an enterprise gateway solving cold latency, vendor rate limits, and multi-database sprawl.

#### Question 2: "In your hybrid retrieval pipeline, how do you handle merging the vector search and keyword search results? Why not just use a linear weighted average of the scores?"
- **What the Interviewer is Evaluating:** Deep understanding of search relevance, score calibration problems, and ranking algorithms.
- **Strong Candidate Response:** Reference [05_TECHNOLOGY_DECISIONS_AND_SYSTEM_DESIGN.md#L96-L122](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/05_TECHNOLOGY_DECISIONS_AND_SYSTEM_DESIGN.md#L96-L122). Explain that cosine distance is bounded $[0, 2]$ (lower is better) while `ts_rank_cd` is unbounded $[0, \infty)$ (higher is better). Explain min-max normalization instability and show how RRF operates purely on ordinal ranks using $\sum \frac{1}{60 + r_m(d)}$.

#### Question 3: "What happens when your primary LLM provider (Groq) experiences an outage or returns HTTP 429?"
- **What the Interviewer is Evaluating:** Production resilience, circuit breaker mechanics, and graceful degradation.
- **Strong Candidate Response:** Reference [02_FEATURE_IMPLEMENTATION_DEEP_DIVES.md#L202-L260](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/02_FEATURE_IMPLEMENTATION_DEEP_DIVES.md#L202-L260). Walk through Tenacity retries with Full Jitter, the 5 consecutive failure threshold, the transition to `OPEN`, the immediate 0ms fail-fast redirect to OpenAI `gpt-4o-mini`, and the 60-second `HALF_OPEN` canary probe.

---

### Mock Round 2: Python Concurrency & Backend Internals (45 Minutes)

#### Question 1: "Walk me through how `asyncio` handles thousands of concurrent requests in a single Python thread. What happens if someone adds a `time.sleep(5)` call in an async route?"
- **What the Interviewer is Evaluating:** Foundational OS and event loop mechanics, non-blocking I/O vs. CPU blocking.
- **Strong Candidate Response:** Reference [03_INTERVIEW_QUESTIONS_AND_ANSWERS.md#L62-L78](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/03_INTERVIEW_QUESTIONS_AND_ANSWERS.md#L62-L78). Explain cooperative multitasking, epoll selectors, task scheduling on `await`, and how `time.sleep(5)` blocks the OS thread, freezing the event loop for all concurrent requests.

#### Question 2: "Why did you implement your rate limiter using a Redis Lua script instead of standard Python code with Redis `INCR`?"
- **What the Interviewer is Evaluating:** Concurrency race conditions, distributed systems, and atomic operations.
- **Strong Candidate Response:** Reference [03_INTERVIEW_QUESTIONS_AND_ANSWERS.md#L173-L190](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/03_INTERVIEW_QUESTIONS_AND_ANSWERS.md#L173-L190). Explain the microsecond race condition across stateless Uvicorn worker pods, why distributed locks add latency, and how Redis Lua scripts execute atomically in Redis's single-threaded event loop using server-side time.

#### Question 3: "We noticed an intermittent `asyncpg.InterfaceError: another operation is in progress` in an async codebase. What causes this and how do you resolve it?"
- **What the Interviewer is Evaluating:** Real-world asyncpg / SQLAlchemy debugging experience.
- **Strong Candidate Response:** Reference [06_TESTING_DEBUGGING_AND_FAILURE_SCENARIOS.md#L50-L63](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/06_TESTING_DEBUGGING_AND_FAILURE_SCENARIOS.md#L50-L63). Explain that an `AsyncSession` holds a single half-duplex connection. Concurrent coroutines via `asyncio.gather` on the same session collide on socket read/write buffers. The fix is running queries sequentially or using two distinct sessions.

---

## 3. Self-Assessment Rubric

Assess your readiness across each technical pillar using this 4-tier rubric:

| Technical Pillar | Level 1: Not Ready | Level 2: Basic | Level 3: Interview-Ready | Level 4: Strong / Staff-Level |
| :--- | :--- | :--- | :--- | :--- |
| **System Architecture** | Cannot explain what SentinelRAG does or confuses it with a simple wrapper. | Can describe the 4 stages but cannot explain the data passed between them. | Explains the complete request flow, file paths, and why components interact as they do. | Proactively discusses failure modes, connection pool tuning, and production scaling trade-offs. |
| **Semantic Caching** | Thinks the cache is a simple Python dictionary or exact string match. | Knows it uses cosine distance, but doesn't know the threshold or how vectors are stored. | Explains the two-stage lookup (SHA-256 + vector scan), base64 packing, and $\le 0.08$ threshold. | Discusses memory growth, lazy dead-key eviction, and migrating to RediSearch HNSW at scale. |
| **Database & pgvector** | Unclear on how vectors are queried in SQL. | Knows pgvector is used, but cannot explain HNSW vs. IVFFlat or cosine distance `<=>`. | Explains HNSW graph indexing, GIN generated columns, and why queries run sequentially on `AsyncSession`. | Explains iterative index scans with multi-tenant filtering, `pool_pre_ping`, and connection leak prevention. |
| **Resilience & Routing** | Mentions retrying requests with standard loops. | Explains try/except fallback, but unaware of Circuit Breaker states or jitter. | Explains CLOSED/OPEN/HALF_OPEN transitions, fail-fast mechanics, and Tenacity Full Jitter formula. | Explains process-local vs. distributed circuit breaker drift across Kubernetes pods and failure isolation. |
| **Async Python & ASGI** | Confuses threads with coroutines or thinks `await` spawns background OS threads. | Understands `async/await`, but cannot explain why blocking calls freeze the worker. | Explains the event loop, cooperative multitasking, `contextvars`, and `Depends` generator cleanup. | Deeply understands epoll, task scheduling, greenlet internals in SQLAlchemy, and socket buffer collisions. |

---

## 4. Prioritized Revision Plan (Final 48 Hours)

### Day 1: Code Tracing & Verbal Articulation (Must-Know)
1. **Morning (2 Hours): Request Flow Tracing**
   - Read [01_ARCHITECTURE_AND_REQUEST_FLOWS.md](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/01_ARCHITECTURE_AND_REQUEST_FLOWS.md).
   - Trace [app/api/v1/chat.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/api/v1/chat.py) line-by-line. Speak aloud each transition (Rate Limiting $\to$ Cache Check $\to$ Hybrid Retrieval $\to$ LLM Router $\to$ Telemetry).
2. **Afternoon (3 Hours): Feature Deep Dives & Math**
   - Review [02_FEATURE_IMPLEMENTATION_DEEP_DIVES.md](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/02_FEATURE_IMPLEMENTATION_DEEP_DIVES.md).
   - Practice writing the RRF formula ($\sum \frac{1}{60 + r}$) and the Full Jitter formula ($U(0, \min(T_{\text{max}}, B \cdot 2^A))$) on paper.
   - Review the Token Bucket Lua script mechanics in [app/core/rate_limiter.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/rate_limiter.py).
3. **Evening (2 Hours): Verbal Mock Practice**
   - Practice Mock Round 1 and Mock Round 2 aloud. Record your voice on your phone, listen back, and eliminate hesitations.

### Day 2: Live Coding Practice & Architectural Defense
1. **Morning (3 Hours): Hands-On Coding Exercises**
   - From [04_CODE_WALKTHROUGHS_AND_CODING_EXERCISES.md](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/04_CODE_WALKTHROUGHS_AND_CODING_EXERCISES.md), code:
     - Exercise 1: Async LRU Cache-Aside helper with TTL.
     - Exercise 2: Reciprocal Rank Fusion from scratch.
     - Exercise 3: 3-State Circuit Breaker with `asyncio.Lock`.
2. **Afternoon (2 Hours): Technology Decisions & Trade-Offs**
   - Review [05_TECHNOLOGY_DECISIONS_AND_SYSTEM_DESIGN.md](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/05_TECHNOLOGY_DECISIONS_AND_SYSTEM_DESIGN.md).
   - Memorize the Dual-System Problem defense for PostgreSQL vs. dedicated vector databases.
   - Review the LangChain vs. custom LiteLLM router comparison.
3. **Evening (2 Hours): Failure Scenarios & Observability**
   - Review [06_TESTING_DEBUGGING_AND_FAILURE_SCENARIOS.md](file:///c:/Users/kiran/Desktop/sentinel-rag/docs/interview_mastery/06_TESTING_DEBUGGING_AND_FAILURE_SCENARIOS.md).
   - Review the troubleshooting steps for `asyncpg.InterfaceError`, Redis socket timeouts, and circuit breaker drift.
   - Get a full 8 hours of sleep before your interview!
