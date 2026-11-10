# SentinelRAG & Backend Engineering Interview Hub

Welcome to **`interview-preparation/`**—your unified, contradiction-free preparation repository covering **SentinelRAG** (Multi-Tenant AI Gateway & Hybrid Retrieval Platform) alongside your **KloudWorx** production experience, **DOit**, live coding drills, and behavioral defense.

> **START HERE:**
> 1. Read **[00-preparation-roadmap.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/00-preparation-roadmap.md)** first (especially Section 3's **Discrepancies & Honesty Guardrails**).
> 2. Follow **[13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md)** for daily structured study (7-day standard track or 5-day fast-track).
> 3. Keep **[11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md)** open for your final 1–2 hour pre-interview warm-up.

---

## The Two-Phase Preparation Pipeline

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│           PHASE 1: SENTINELRAG DEEP MASTERY (DAYS 1 – 4)                     │
│  Architecture, Redis Lua, Semantic Cache, pgvector HNSW, RRF, CB Routing     │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
  ├── 01–04                          │ Overview, Architecture, Workflows, Core Concepts
  ├── 05-technology-deep-dives/      │ FastAPI, Postgres/pgvector, Redis/Lua, LLMs, Infra
  ├── 06-implementation-deep-dives/  │ Module-by-module code walkthroughs & edge cases
  └── 07–10                          │ Layered Q&A Bank, Scenarios, Trade-Offs, Follow-Ups
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│           PHASE 2: RESUME, LIVE CODING & MOCKS (DAYS 5 – 7)                  │
│  KloudWorx Production Systems, DOit, Python Drills, STAR & Timed Mocks       │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
  ├── 14-resume-and-behavioral-defense.md  │ KloudWorx, DOit, Chemistry pivot, 4 STAR blueprints
  ├── 07-interview-question-bank/E-live-coding-exercises.md │ 5 timed Python coding drills
  ├── 12-mock-interview.md                 │ 4 full 45-min mock rounds + 20-pt grading rubric
  └── 11-quick-revision.md                 │ Final 2-hour cheat sheet & 10-question self-check
```

---

## Study Schedule & Estimated Time Allocations

Full execution instructions live in **[13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md)** (`~21 hours` over 7 days at `~3.0 hrs/day`, or `~16.5 hours` on the compressed 5-day track).

- **Standard Daily 3-Hour Cadence:** `15 min` prior-day active recall $\to$ `75 min` concept/code reading $\to$ `45 min` whiteboard/coding drill $\to$ `45 min` spoken Q&A & cross-question defense *(Day 1 puts `90 min` into reading with no prior-day warm-up; Day 7 allocates `90 min` to timed mocks)*.

| Track / Day | Focus Area | Primary Files | Est. Time |
| :--- | :--- | :--- | :---: |
| **Day 1** *(Phase 1)* | Honesty Audit, SentinelRAG Topology, Lifecycle & FastAPI/AsyncIO | `00`, `01`, `02`, `03`, `05/01`, `06/01`, `07/A` | **3.0 hrs** |
| **Day 2** *(Phase 1)* | Redis Lua Token Bucket & Two-Tier Semantic Cache | `04`, `05/03`, `06/02`, `10` (Chains 1, 3) | **3.0 hrs** |
| **Day 3** *(Phase 1)* | PostgreSQL `pgvector` HNSW, GIN FTS & RRF ($k=60$) | `05/02`, `06/03`, `07/B`, `07/E` (Ex 2), `10` (Chain 2) | **3.0 hrs** |
| **Day 4** *(Phase 1)* | LLM Router, Circuit Breaker, Infra, Scenarios & Trade-Offs | `05/04`, `05/05`, `06/04`, `08`, `09`, `07/E` (Ex 3), `10` (4–6) | **3.0 hrs** |
| **Day 5** *(Phase 2)* | KloudWorx Distributed Systems (Slim $\to$ FastAPI, SQS, S3, Stripe) + Coding | `14` (Secs 1–5), `07/E` (Ex 1, 4) | **3.0 hrs** |
| **Day 6** *(Phase 2)* | MySQL ESR/Deadlocks, Cache-Aside, ES `nested`, DOit + Coding | `14` (Secs 6–8), `07/C`, `07/D`, `07/E` (Ex 5) | **3.0 hrs** |
| **Day 7** *(Phase 2)* | Behavioral STAR, M.Sc. Chemistry Pivot, Timed Mock & Gap Patching | `14` (Secs 9–11), `12`, `11` | **3.0 hrs** |
| **Pre-Interview** | Night-before walkthrough (`45m`) + interview-morning warm-up (`45–60m`) | `11-quick-revision.md` | **1.5–1.75 hrs** |

> **Compressed 5-Day Fast-Track (`~16.5 hrs` total):** Day 1 (`3.0h`) $\to$ Day 2 (`3.5h`: Cache + Postgres/RRF) $\to$ Day 3 (`3.5h`: Router/Infra/Scenarios) $\to$ Day 4 (`3.5h`: All KloudWorx/DOit + Coding) $\to$ Day 5 (`3.0h`: STAR + Mock + Final Cheat Sheet). Details in [13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md#compressed-5-day-fast-track-mapping).

---

## Complete Repository Map (`27` Files)

### Foundations & System Design
- **[00-preparation-roadmap.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/00-preparation-roadmap.md)** — Study map, audit of original `notes/` sources, and verified fixes for conflicting numbers/claims.
- **[01-project-overview.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/01-project-overview.md)** — Problem space, 30s & 2m pitches, core components, and implemented vs. planned scope.
- **[02-project-architecture.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/02-project-architecture.md)** — Topology diagram, Starlette LIFO middleware execution order, service boundaries, and RFC 7807 contract.
- **[03-end-to-end-workflow.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/03-end-to-end-workflow.md)** — Complete `POST /api/v1/chat/completions` execution trace across Cache Hit, Primary Success, and Breaker Failover.
- **[04-core-concepts.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/04-core-concepts.md)** — First-principles explanations (`asyncio`/`contextvars`, Lua Token Bucket, cosine distance, HNSW + `tsvector` GIN, RRF $k=60$, Circuit Breaker + Full Jitter).

### Technology Deep Dives (`05-technology-deep-dives/`)
- **[01-fastapi-asgi-and-asyncio.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/01-fastapi-asgi-and-asyncio.md)** — ASGI vs WSGI, `uvloop`, `async def` vs `def`, lifespan, `Depends` `yield` DAG, Pydantic v2.
- **[02-postgresql-pgvector-and-fts.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/02-postgresql-pgvector-and-fts.md)** — `asyncpg`, PgBouncer `statement_cache_size=0`, HNSW vs IVFFlat, persisted `tsvector` GIN, sequential `AsyncSession`.
- **[03-redis-lua-and-semantic-caching.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/03-redis-lua-and-semantic-caching.md)** — Atomic Lua (`EVALSHA`, `TIME`), `volatile-lru`, IEEE-754 Base64 serialization, lazy `SREM` cleanup, RediSearch upgrade trigger.
- **[04-llm-routing-resilience-and-tokens.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/04-llm-routing-resilience-and-tokens.md)** — LiteLLM cascade, Full Jitter vs Equal Jitter, `CircuitBreaker` `asyncio.Lock` + `_probe_in_flight` race, `tiktoken`.
- **[05-observability-testing-and-infra.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/05-technology-deep-dives/05-observability-testing-and-infra.md)** — `structlog`, Prometheus telemetry, `44` mocked `pytest-asyncio` tests, multi-stage `uv` Dockerfile (`UID 10001`), Nginx, CI/CD.

### Implementation Walkthroughs (`06-implementation-deep-dives/`)
- **[01-app-lifecycle-and-middleware.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/01-app-lifecycle-and-middleware.md)** — `config.py`, `logging.py`, `database.py`, `session.py`, `redis.py`, `exceptions.py`, `telemetry.py`, `main.py`.
- **[02-rate-limiter-and-semantic-cache.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/02-rate-limiter-and-semantic-cache.md)** — Annotated `TOKEN_BUCKET_LUA`, `TokenBucketLimiter`, `RedisSemanticCache`, and cache endpoints.
- **[03-hybrid-search-and-rrf.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/03-hybrid-search-and-rrf.md)** — `DocumentChunk`, Alembic migrations, `/api/v1/documents/batch`, `SearchRepository`, RRF fusion, `/api/v1/search/hybrid`.
- **[04-resilient-router-chat-and-analytics.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/04-resilient-router-chat-and-analytics.md)** — `CircuitBreaker`, Tenacity retryer, `ResilientLLMRouter`, `TokenAnalyticsTracker`, `chat.py`, `dashboard/app.py`.

### Question Banks & Practice (`07-interview-question-bank/` & `08`–`10`)
- **[A-project-overview-and-system-flow.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/A-project-overview-and-system-flow.md)** — Pitch, ownership boundaries, request lifecycle, and endpoints.
- **[B-architecture-and-design.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/B-architecture-and-design.md)** — Trust zones, single-store Postgres vs Pinecone/ES, solved challenges, 100x scaling blueprint.
- **[C-technology-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/C-technology-questions.md)** — Layered FastAPI, `contextvars`, `pgvector`, FTS, Redis Lua, Tenacity/`tiktoken`, and Docker/Nginx Q&As.
- **[D-implementation-and-operations.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/D-implementation-and-operations.md)** — Code-level deep Q&As on sequential search, float packing, RRF tie-breaking, CB cascade, sessions, testing.
- **[E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md)** — 5 production Python coding drills (Async TTL LRU Cache, RRF, Async Circuit Breaker, Full Jitter Retry, Token Bucket).
- **[08-scenario-based-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/08-scenario-based-questions.md)** — Outages, cache stampedes, p99 latency runbook, 10x traffic spike, `asyncpg.InterfaceError`, stale post-update answers.
- **[09-design-decisions-and-tradeoffs.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/09-design-decisions-and-tradeoffs.md)** — Alternatives rejected, current implementation shortcuts, and migration thresholds.
- **[10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md)** — 6 multi-level interviewer cross-questioning chains.

### Revision, Mocks, Schedule & Phase 2 Resume Defense
- **[11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md)** — Final 2-hour high-density cheat sheet, verified numbers table, 8 code traps, Phase 2 flashcards, 10 self-checks.
- **[12-mock-interview.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/12-mock-interview.md)** — 4 timed 45-minute mock interview rounds with interviewer probe intent, trap answers, and 20-point rubric.
- **[13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md)** — Actionable 7-day plan (preserving [MASTER_STUDY_SCHEDULE.md](file:///c:/Users/kiran/Desktop/notes/MASTER_STUDY_SCHEDULE.md)) plus 5-day fast-track compression.
- **[14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md)** — Phase 2: KloudWorx production systems (FastAPI/Slim, SQS, S3, Stripe, MySQL ESR, Redis Cache-Aside, ES `nested`), `DOit`, M.Sc. Chemistry transition, 4 STAR blueprints, and interviewer questions.

---

## Quick Decision Guide: What Should I Open Right Now?

| Your Immediate Situation | Open This First |
| :--- | :--- |
| **Starting fresh with 5–7 days before interviews** | [13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md) $\to$ [00-preparation-roadmap.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/00-preparation-roadmap.md) |
| **Interview starts in 1–2 hours** | [11-quick-revision.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/11-quick-revision.md) |
| **Practicing speaking & handling cross-questioning** | [10-follow-up-and-cross-questions.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/10-follow-up-and-cross-questions.md) $\to$ [12-mock-interview.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/12-mock-interview.md) |
| **Preparing for coding / live debugging rounds** | [E-live-coding-exercises.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/07-interview-question-bank/E-live-coding-exercises.md) $\to$ [12-mock-interview.md (Round 2)](file:///c:/Users/kiran/Desktop/notes/interview-preparation/12-mock-interview.md) |
| **Preparing for KloudWorx / HR / EM behavioral rounds** | [14-resume-and-behavioral-defense.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/14-resume-and-behavioral-defense.md) |
| **Checking exact verbatim source lines or Docker/Nginx configs** | [archive/supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md) *(read-only code archive)* |

---

## Relationship to Older Folders in `notes/`

- All original folders ([001curriculum/](file:///c:/Users/kiran/Desktop/notes/001curriculum/PART1-project-reconnaissance.md), [002phase-1-implementation/](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/01-project-overview.md), [003phase-2-interview-prep/](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/05-tradeoffs-and-system-design.md), [004resume-deepdive/](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md), [interview_mastery/](file:///c:/Users/kiran/Desktop/notes/interview_mastery/00_PROJECT_MASTER_GUIDE.md), and [archive/](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md)) are **preserved untouched** as read-only reference material.
- **`interview-preparation/`** is your primary active study workspace—every default value, middleware order, endpoint path, and project claim here is reconciled directly against [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md) and [kiranteja-ai-engineer.pdf](file:///c:/Users/kiran/Desktop/notes/resume/kiranteja-ai-engineer.pdf).

