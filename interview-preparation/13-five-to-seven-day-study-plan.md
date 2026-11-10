# 13. 5-to-7 Day Active Study & Revision Plan

This plan preserves the two-phase structure of [MASTER_STUDY_SCHEDULE.md](file:///c:/Users/kiran/Desktop/notes/MASTER_STUDY_SCHEDULE.md)—**Phase 1 (Days 1–4: SentinelRAG Deep Mastery)** followed by **Phase 2 (Days 5–7: KloudWorx Resume Defense, Live Coding, Behavioral & Full Mock)**—and upgrades passive reading into **timed active recall, whiteboard sketching, and spaced-repetition drills**.

> **Daily Time Cadence (~3 Hours/Day):**
> - **0:00–0:15 — Spaced Repetition Recall:** Answer 5–8 questions from *yesterday's* topic aloud without notes before opening any file.
> - **0:15–1:30 — Focused Concept & Code Study:** Read the exact assigned sections inside `interview-preparation/`.
> - **1:30–2:15 — Active Production (Whiteboard / Coding):** Sketch diagrams from memory, trace execution line-by-line, or implement a module cleanly.
> - **2:15–3:00 — Out-Loud Q&A & Cross-Question Defense:** Practice layered answers (20–40s Short Answer first!) and grade weak spots.

---

## Quick Schedule Overview

| Day | Phase & Theme | Primary `interview-preparation/` Guides | Practical Output / Milestone |
| :--- | :--- | :--- | :--- |
| **Day 1** | **Phase 1:** SentinelRAG Overview, Lifecycle & FastAPI/AsyncIO | [00](file:///c:/Users/kiran/Desktop/notes/interview-preparation/00-preparation-roadmap.md), [01](file:///c:/Users/kiran/Desktop/notes/interview-preparation/01-project-overview.md), [02](file:///c:/Users/kiran/Desktop/notes/interview-preparation/02-project-architecture.md), [03](file:///c:/Users/kiran/Desktop/notes/interview-preparation/03-end-to-end-workflow.md), [05/01](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/01-fastapi-asgi-and-asyncio.md), [06/01](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/01-app-lifecycle-and-middleware.md) | 2-min pitch fluent; draw middleware order + request lifecycle from blank paper |
| **Day 2** | **Phase 1:** Redis Lua Token Bucket & Semantic Caching | [04](file:///c:/Users/kiran/Desktop/notes/interview-preparation/04-core-concepts.md), [05/03](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/03-redis-lua-and-semantic-caching.md), [06/02](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md), [10 (Chains 1 & 3)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md) | Re-implement Token Bucket Lua logic + explain SHA-256 + Base64 cosine scan cleanly |
| **Day 3** | **Phase 1:** PostgreSQL `pgvector` HNSW, FTS & RRF Fusion | [05/02](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/02-postgresql-pgvector-and-fts.md), [06/03](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md), [07/B](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/B-architecture-and-design.md), [10 (Chain 2)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md) | Explain `AsyncSession` sequential constraint, HNSW vs IVFFlat, RRF ($k=60$) code |
| **Day 4** | **Phase 1:** Resilient Router, Telemetry, Infra & Trade-Offs | [05/04](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/04-llm-routing-resilience-and-tokens.md), [05/05](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/05-observability-testing-and-infra.md), [06/04](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/04-resilient-router-chat-and-analytics.md), [08](file:///c:/Users/kiran/Desktop/notes/interview-preparation/08-scenario-based-questions.md), [09](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md) | Draw 3-state CB FSM + solve all 7 outage/scaling scenarios under time pressure |
| **Day 5** | **Phase 2:** KloudWorx Distributed Systems & Live Coding | [14 (Secs 1–5)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md), [07/E (Drills 1–3)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md) | Defend SQS idempotency, S3 pre-signed/magic bytes, Stripe HMAC, Slim $\to$ FastAPI |
| **Day 6** | **Phase 2:** MySQL/ES/Cache-Aside, DOit & Live Coding | [14 (Secs 6–8)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md), [07/C](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/C-technology-questions.md), [07/D](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/D-implementation-and-operations.md), [07/E (Drills 4–5)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md) | Complete remaining coding drills + MySQL ESR / deadlock / ES `nested` defense |
| **Day 7** | **Phase 2:** Behavioral STAR, Full 4-Round Mock & Revision | [14 (Secs 9–11)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md), [12-mock-interview.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/12-mock-interview.md), [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md) | Finish all 4 mock rounds out loud; verify every KloudWorx metric & Chemistry pivot |

---

## Detailed Daily Execution Guide (7-Day Standard Track)

### Day 1 — Gateway Foundation, Honesty Audit & Request Lifecycle
- **Learning Objectives:** Separate verified SentinelRAG code facts from old-note myths; master ASGI vs WSGI, event-loop starvation, Starlette LIFO middleware ordering, lifespan resource management, and RFC 7807 error contracts.
- **Read:**
  1. [00-preparation-roadmap.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/00-preparation-roadmap.md) *(Section 3 discrepancy table — memorize every correction)*
  2. [01-project-overview.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/01-project-overview.md) & [02-project-architecture.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/02-project-architecture.md)
  3. [03-end-to-end-workflow.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/03-end-to-end-workflow.md)
  4. [05/01-fastapi-asgi-and-asyncio.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/01-fastapi-asgi-and-asyncio.md) & [06/01-app-lifecycle-and-middleware.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/01-app-lifecycle-and-middleware.md)
- **Active Recall & Drill:**
  - Close notes and draw the 3 execution branches of `POST /api/v1/chat/completions` (Cache Hit, Primary Success, Breaker Failover) and the 3-middleware execution order.
  - Answer Q1–Q4 from [07/A-project-overview-and-system-flow.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/A-project-overview-and-system-flow.md) out loud with a 40-second timer for Layer 1.

---

### Day 2 — Redis Lua Rate Limiting & Two-Tier Semantic Caching
- **Warm-Up (15 min):** Recite the exact middleware order and what breaks if an `async def` handler calls `requests.get()`.
- **Learning Objectives:** Explain why single-threaded Redis Lua guarantees atomicity without multi-key distributed locks; trace `TOKEN_BUCKET_LUA` (`TIME`, lazy capacity refill, TTL renewal, Fail-Open); master SHA-256 L1 vs `SMEMBERS` + pipelined `HGETALL` + Base64 IEEE-754 float L2 scan (`<= 0.08`), ghost-key `SREM`, and the RediSearch upgrade path.
- **Read:**
  1. [04-core-concepts.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/04-core-concepts.md) *(Token Bucket & Semantic Similarity sections)*
  2. [05/03-redis-lua-and-semantic-caching.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/03-redis-lua-and-semantic-caching.md)
  3. [06/02-rate-limiter-and-semantic-cache.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md)
- **Active Recall & Cross-Questions:**
  - Drill **Chain 1 (Semantic Caching)** and **Chain 3 (Lua Rate Limiting)** in [10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md).
  - Practice explaining *why* Base64 encoding is mandatory when `decode_responses=True` is set on the Redis client.

---

### Day 3 — Single-Store Hybrid Search, `pgvector` HNSW & RRF
- **Warm-Up (15 min):** Explain ghost-key `SREM` cleanup and why Redis `TIME` beats application UTC timestamps in the rate-limit Lua script.
- **Learning Objectives:** Explain PostgreSQL + `pgvector` (`<=>`, HNSW `m=16, ef_construction=64` vs IVFFlat) + persisted `tsvector` GIN + `websearch_to_tsquery`/`ts_rank_cd`; justify why `SearchRepository` cannot `asyncio.gather` on one `AsyncSession`; implement RRF (`k=60`) with `(-rrf_score, -dense_score, -sparse_score)` tie-breaking from scratch.
- **Read:**
  1. [05/02-postgresql-pgvector-and-fts.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/02-postgresql-pgvector-and-fts.md)
  2. [06/03-hybrid-search-and-rrf.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md)
  3. [07/B-architecture-and-design.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/B-architecture-and-design.md)
- **Active Recall & Live Coding:**
  - Write **Exercise 2 (Reciprocal Rank Fusion)** from [07/E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md) in Python without looking at the solution.
  - Drill **Chain 2 (Single-Store Postgres & pgvector)** in [10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md).

---

### Day 4 — LLM Routing, Resilience, Observability & Scenarios
- **Warm-Up (15 min):** From memory, compare HNSW vs IVFFlat recall/RAM/rebuild behavior and state the RRF formula.
- **Learning Objectives:** Trace Groq (`3x`) $\to$ OpenAI (`2x`) $\to$ Mistral (`1x`) failover; explain `CircuitBreaker` (`5` failures, `60s` recovery, `4.0s` timeout) plus the `_probe_in_flight` concurrency improvement; trace Tenacity Full Jitter math, `tiktoken` pricing/special-token bug, Prometheus 4 metrics, multi-stage `uv` Dockerfile (`UID 10001`, `--workers 1`), Nginx `proxy_buffering off`, and 44 mocked tests.
- **Read:**
  1. [05/04-llm-routing-resilience-and-tokens.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/04-llm-routing-resilience-and-tokens.md) & [05/05-observability-testing-and-infra.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/05-observability-testing-and-infra.md)
  2. [06/04-resilient-router-chat-and-analytics.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/04-resilient-router-chat-and-analytics.md)
  3. [08-scenario-based-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/08-scenario-based-questions.md) & [09-design-decisions-and-tradeoffs.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md)
- **Active Recall & Drill:**
  - Drill **Chains 4, 5 & 6** in [10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md).
  - Write **Exercise 3 (3-State Async Circuit Breaker)** from [07/E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md) from scratch.

---

### Day 5 — KloudWorx Distributed Production Systems & Live Coding
- **Warm-Up (15 min):** Solve 3 random scenarios from [08-scenario-based-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/08-scenario-based-questions.md) (e.g., `asyncpg.InterfaceError`, stale answers after document update, 10x token spike).
- **Learning Objectives:** Defend every KloudWorx distributed systems bullet accurately—Strangler Fig PHP Slim $\to$ FastAPI migration, AWS SQS producer/consumer idempotency + DLQ, S3 direct pre-signed URLs + magic bytes validation, Stripe HMAC-SHA256 + timestamp replay protection + fast 200 ACK, and JWT/RBAC `Depends`.
- **Read:**
  1. [14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md) *(Sections 1–5)*
- **Active Recall & Live Coding:**
  - Draw the KloudWorx event-driven upload & webhook flow on paper.
  - Solve **Exercises 1 & 4 (Async TTL LRU Cache & Full Jitter Retry)** in [07/E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md).

---

### Day 6 — Database Internals, Elasticsearch, DOit & Final Coding Drill
- **Warm-Up (15 min):** Recite SQS idempotency (`UUIDv4` + Redis `SET NX EX`) + Stripe HMAC replay window without notes.
- **Learning Objectives:** Master MySQL Slow Query Log, `EXPLAIN ANALYZE`, B+Tree **ESR** composite index ordering, stored procs vs SQLAlchemy trade-offs, InnoDB deadlocks, Cache-Aside `DEL` vs `UPDATE`, Elasticsearch `nested` vs `object`, **DOit** multi-workspace architecture, and remaining coding/implementation Q&As.
- **Read:**
  1. [14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md) *(Sections 6–8)*
  2. [07/C-technology-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/C-technology-questions.md) & [07/D-implementation-and-operations.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/D-implementation-and-operations.md) *(spot-check weak answers only)*
- **Active Recall & Live Coding:**
  - Solve **Exercise 5 (In-Memory Token Bucket)** from [07/E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md).
  - Practice **Exercise**: Given SQL `WHERE tenant_id = ? AND status = ? AND created_at >= ? ORDER BY priority DESC`, design the composite index and justify column order via ESR.

---

### Day 7 — Behavioral STAR, M.Sc. Chemistry Pivot & Full Mock Simulation
- **Warm-Up (15 min):** Run the 10-question rapid self-check in [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md).
- **Execution Plan:**
  1. **Hour 1 (Behavioral Calibration):** Review Sections 9–11 of [14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md). Audit each STAR story so every number/incident reflects your true personal KloudWorx experience rather than memorized placeholders. Practice your M.Sc. Chemistry $\to$ Backend/AI Engineer narrative in 45 seconds.
  2. **Hours 2–3 (Timed Mock Interview):** Complete at least **Round 1, Round 3, and Round 4** (or all 4 rounds at 30 mins each) from [12-mock-interview.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/12-mock-interview.md) out loud, scoring yourself with the 20-point rubric.
  3. **Final 30 Mins (Patch Gaps):** Re-read only the sections where you hesitated in the mock, then keep [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md) open for interview day.

---

## Compressed 5-Day Fast-Track Mapping (If Time Is Short)

If an interview is **5 days away**, combine Days 2+3 and Days 5+6 cleanly without dropping active mocks:

| Fast-Track Day | Combined Scope | Core Docs to Prioritize |
| :--- | :--- | :--- |
| **Day 1** | SentinelRAG Overview, Middleware Order & AsyncIO | `01`, `02`, `03`, `05/01`, `06/01` |
| **Day 2** | Redis Lua + Semantic Cache + Postgres `pgvector` HNSW + RRF | `04`, `05/02`, `05/03`, `06/02`, `06/03`, `07/E (Ex 2)` |
| **Day 3** | LLM Router, CB, Infra, Scenarios & Trade-Offs | `05/04`, `05/05`, `06/04`, `08`, `09`, `10` |
| **Day 4** | KloudWorx Architecture (SQS/S3/Stripe/MySQL/Redis/ES) + DOit + Coding | `14 (All Technical)`, `07/E (Ex 1, 3, 5)` |
| **Day 5** | Behavioral STAR, Chemistry Pivot, Timed Mock & Final Cheat Sheet | `14 (STAR/Behavioral)`, `12-mock-interview.md`, `11-quick-revision.md` |

---

## Final Night-Before & Interview-Morning Protocol

- **Night Before (45 mins):** Do **not** cram new material. Run through Sections 1–4 of [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md) once, sleep properly.
- **Morning Of (45–60 mins):**
  1. Speak your **30s SentinelRAG pitch**, **90s Resume pitch**, and **M.Sc. Chemistry transition** aloud twice.
  2. Draw the **5-box SentinelRAG diagram** and **KloudWorx async upload/webhook flow** once cleanly.
  3. Run the **10 Pass/Fail Self-Check** questions at the bottom of [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md).
  4. Keep your 3 tailored questions for the interviewer ready ([14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md#11-high-signal-questions-to-ask-the-interviewer)).

