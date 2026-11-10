# 00. Interview Preparation Roadmap & Repository Audit

This `interview-preparation/` repository transforms the raw documentation in `notes/` into a focused, contradiction-free, active-learning resource for technical interviews on **SentinelRAG** (and your broader backend engineering profile).

Every technical claim, symbol name, route path, and default parameter in these guides has been cross-checked directly against the full source code archived in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md) and the implementation walkthroughs in [001curriculum/](file:///c:/Users/kiran/Desktop/notes/001curriculum/PART1-project-reconnaissance.md).

---

## 1. How This Repository Is Organized

```
notes/interview-preparation/
├── 00-preparation-roadmap.md               # This file: study map, source audit, discrepancy log
├── 01-project-overview.md                  # Problem space, elevator pitches, verified vs. planned features
├── 02-project-architecture.md              # Topology, verified middleware order, contracts, trust boundaries
├── 03-end-to-end-workflow.md               # Step-by-step request lifecycle, 3 execution paths, all routes
├── 04-core-concepts.md                     # First-principles CS/IR/distributed systems mental models
├── 05-technology-deep-dives/               # Deep technical mastery of every stack layer
│   ├── 01-fastapi-asgi-and-asyncio.md
│   ├── 02-postgresql-pgvector-and-fts.md
│   ├── 03-redis-lua-and-semantic-caching.md
│   ├── 04-llm-routing-resilience-and-tokens.md
│   └── 05-observability-testing-and-infra.md
├── 06-implementation-deep-dives/           # Module-by-module code walkthroughs & edge-case analysis
│   ├── 01-app-lifecycle-and-middleware.md
│   ├── 02-rate-limiter-and-semantic-cache.md
│   ├── 03-hybrid-search-and-rrf.md
│   └── 04-resilient-router-chat-and-analytics.md
├── 07-interview-question-bank/             # Layered Q&As (Short -> Detailed -> Project -> Follow-up -> Example)
│   ├── A-project-overview-and-system-flow.md
│   ├── B-architecture-and-design.md
│   ├── C-technology-questions.md
│   ├── D-implementation-and-operations.md
│   └── E-live-coding-exercises.md
├── 08-scenario-based-questions.md          # Realistic outage, traffic spike, and debugging scenarios
├── 09-design-decisions-and-tradeoffs.md    # Architectural choices, limitations, and migration thresholds
├── 10-follow-up-and-cross-questions.md     # Progressive 5-7 level interviewer cross-questioning chains
├── 11-quick-revision.md                    # Final 2-hour high-density cheat sheet & gotcha list
├── 12-mock-interview.md                    # Realistic 4-round mock interview simulation scripts
├── 13-five-to-seven-day-study-plan.md      # Actionable 7-day (and 5-day fast-track) active study plan
└── 14-resume-and-behavioral-defense.md     # Phase 2: KloudWorx experience, DOit, Chemistry pivot, STAR
```

---

## 2. Source Repository Audit (Read-Only Reference Map)

Before creating this directory, every directory in `notes/` was audited to separate verified, interview-ready material from redundant drafts and historical logs:

| Original Source | Classification | What It Contains & Why It Was Reorganized |
| :--- | :--- | :--- |
| [archive/supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md) | **Essential Ground Truth (Code)** | 7,749 lines containing the full source code for Days 1–7, tests, Dockerfile, Nginx configs, CI/CD workflow, plus a duplicated trace and 56-question bank. Too huge for active study, but indispensable as code verification. |
| [001curriculum/](file:///c:/Users/kiran/Desktop/notes/001curriculum/PART1-project-reconnaissance.md) | **Essential Learning Reference** | Pedagogical Day 1–7 modules with code excerpts and explicit callouts where earlier daily logs diverged from code. Note: Day 1 `database.py` excerpt differs slightly from the final `supplementary.md` code. |
| [002phase-1-implementation/](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/01-project-overview.md) | **Supporting Reference (Mixed)** | Good high-level architecture and 10-point concepts, but contains several path/parameter inaccuracies (e.g., `/v1/documents/ingest`, `DELETE /v1/cache/invalidate`, `failure_threshold=3`) fixed here. |
| [003phase-2-interview-prep/](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/05-tradeoffs-and-system-design.md) | **Directly Useful for Q&A (With Caveats)** | Strong 56-Q&A technology bank (`07`), trade-offs (`05`), security (`06`), and whiteboard drills (`08`). However, `05` includes a fictional "500 B2B customers / $180k/mo" STAR story that contradicts your actual resume. |
| [004resume-deepdive/](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md) | **Essential for Phase 2 (Resume)** | Detailed technical defense of your ~3 years at KloudWorx (SQS, S3, MySQL, Redis, Stripe, Elasticsearch) and 4 STAR stories. Superseded [RESUME_AND_INTERVIEW_GUIDE.md](file:///c:/Users/kiran/Desktop/notes/RESUME_AND_INTERVIEW_GUIDE.md). |
| [MASTER_STUDY_SCHEDULE.md](file:///c:/Users/kiran/Desktop/notes/MASTER_STUDY_SCHEDULE.md) | **Foundation for Study Plan** | Defines your 7-day, 2-phase schedule (Days 1–4 SentinelRAG, Days 5–7 KloudWorx/Behavioral). Preserved and expanded with active recall in [13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md). |
| [AI_BUILDER_TO_MASTER_BLUEPRINT.md](file:///c:/Users/kiran/Desktop/notes/AI_BUILDER_TO_MASTER_BLUEPRINT.md) | **Supporting Framing** | Good "tactical scar" drills, though some claims (e.g., `FT.SEARCH`, `struct.pack('<1536f')`) do not match the actual `cache.py` implementation. |
| [interview_mastery/](file:///c:/Users/kiran/Desktop/notes/interview_mastery/00_PROJECT_MASTER_GUIDE.md) | **Mostly Redundant (Unique Gems Extracted)** | Earlier draft largely superseded by `002`/`003`, EXCEPT `04_CODE_WALKTHROUGHS_AND_CODING_EXERCISES.md` (5 live coding exercises) and `06_TESTING_DEBUGGING_AND_FAILURE_SCENARIOS.md`, whose unique material is integrated here. |
| [archive/daily_logs/](file:///c:/Users/kiran/Desktop/notes/archive/daily_logs/DAY_0_ARCHITECTURE_PRD_AND_ROADMAP.md) | **Descriptive / Historical (Skip)** | Chronological build logs with some PRD wish-list items (e.g., prompt sanitizer) never implemented in final code. |
| [RESUME_AND_INTERVIEW_GUIDE.md](file:///c:/Users/kiran/Desktop/notes/RESUME_AND_INTERVIEW_GUIDE.md) | **Superseded / Redundant** | Early draft mixing MD5 cache claims and 3-failure breaker thresholds; superseded by `004resume-deepdive/` and this repository. |

---

## 3. Critical Discrepancies & Honesty Guardrails (Read Before Any Interview)

Across the raw notes, several conflicting details exist. Memorizing the wrong version—or presenting unverified/fictional numbers as real production metrics—creates severe credibility risk if an interviewer probes deeper. Use this table as your single source of truth:

| Topic | Conflicting Claims in Old Notes | Verified Reality in Code & Resume | How to State It Accurately |
| :--- | :--- | :--- | :--- |
| **Project Status & Naming** | Called `SentinelRack` in `001curriculum`; presented as a live enterprise deployment ("500 B2B customers", "$180k/mo spend") in `003/05`. | Official resume ([kiranteja-ai-engineer.pdf](file:///c:/Users/kiran/Desktop/notes/resume/kiranteja-ai-engineer.pdf)) lists **SentinelRAG** as an **In-Progress GitHub Project** (not a KloudWorx production workload). | Present SentinelRAG as an architectural/engineering project you built to master enterprise AI infrastructure—never fabricate live employer customers or production incidents around it. |
| **Semantic Cache Lookup** | Some files claim `MD5` hash (`RESUME_AND_INTERVIEW_GUIDE.md`, `004`) or RediSearch `FT.SEARCH` (`AI_BUILDER_TO_MASTER_BLUEPRINT.md`, `MASTER_STUDY_SCHEDULE.md`). | Actual `app/services/cache.py` uses **SHA-256** (`hashlib.sha256`), **`SMEMBERS` + pipelined `HGETALL`**, and an in-Python cosine similarity loop—NOT RediSearch `FT.SEARCH`. | Explain the two-tier SHA-256 + Python cosine scan clearly, and cite RediSearch `FT.SEARCH` HNSW as the **planned scaling upgrade** beyond ~5,000 keys/tenant. |
| **Float Serialization** | `MASTER_STUDY_SCHEDULE.md` mentions `struct.pack('<1536f')`. | Actual `serialize_embedding()` calls `struct.pack(f"{len(vector)}f", *vector)` (native endianness) + Base64 ASCII encoding. | State `struct.pack(f"{len(vector)}f", *vector)` + Base64 (needed because `RedisManager` sets `decode_responses=True`), and note explicit `<` little-endian is good practice across heterogeneous nodes. |
| **Middleware Execution Order** | `PART1` and `interview_mastery/01` say Rate Limiter runs *before* Correlation middleware. | `app/main.py` registers `TokenBucketRateLimitMiddleware` via `add_middleware` first, then `CORSMiddleware`, then `@app.middleware("http") correlation_id_and_timing_middleware`. In Starlette, **last added runs first**. | State the true order: `correlation_id_and_timing_middleware` $\to$ `CORSMiddleware` $\to$ `TokenBucketRateLimitMiddleware`. That is also why `TokenBucketRateLimitMiddleware` can read `request.state.request_id`! |
| **Circuit Breaker & Retry Defaults** | `04-codebase-and-trace.md` and `RESUME_AND_INTERVIEW_GUIDE.md` sometimes state 3 failures / 30s / 15s timeout. | `app/core/config.py` defaults: **`CIRCUIT_BREAKER_FAILURE_THRESHOLD = 5`**, **`RECOVERY_TIME_SECONDS = 60`**, **`TIMEOUT_SECONDS = 4.0`**. `router.py` runs `max_attempts=3` on primary, `max_attempts=2` on fallback (`gpt-4o-mini`), then 1 unretried call on tertiary (`mistral-small-latest`). | Use the verified `config.py` defaults (`5` consecutive failures, `60s` cooldown, `4.0s` per-attempt timeout, `3` primary / `2` fallback attempts). |
| **Rate Limiter Defaults** | Some docs cite `60` req/min (`1/sec`); others cite `100` capacity (`10/sec`). | `TokenBucketLimiter.check()` method signature defaults to `(60, 1.0)`, **but** `app/main.py` passes `settings.RATE_LIMIT_DEFAULT_CAPACITY` (`100`) and `REFILL_RATE_PER_SEC` (`10`). | Cite `100` capacity / `10/sec` refill from `config.py` (and note it's configurable via env vars). |
| **API Route Paths** | `04-codebase-and-trace.md` says `/v1/documents/ingest` and `DELETE /v1/cache/invalidate`. | Tests and routers (`documents.py`, `cache.py`, `search.py`, `chat.py`) use `/api/v1/documents/batch`, `/api/v1/search/hybrid`, `/api/v1/cache/evaluate` (`/set`, `POST /invalidate`), and `/api/v1/chat/completions`. | Use the exact `/api/v1/...` paths tested in `tests/`. |
| **Benchmark Metrics (e.g., 54.2% savings, 44/44 tests)** | Quoted as "measured production performance" in several docs. | The test suite (`44` tests across `12` test modules) uses **`AsyncMock`, `FakeRedis`, and mocked `litellm.acompletion`** (`ASGITransport`), verifying logic and in-memory latency (`<25ms` mocked cache hit). Live production numbers (`54.2%`, `12–18ms`, `850–1400ms`) are **unverified / target SLOs**. | Distinguish clearly between **target SLOs / mocked test assertions** and **unverified production metrics**. Never claim mocked test results came from live enterprise production traffic. |
| **Unimplemented PRD Items** | `DAY_0` PRD lists regex prompt sanitizer; `Q35` mentions `allowed_special=set()`; `Day 7` discusses SSE streaming. | No prompt sanitizer module exists in `app/`; `token_analytics.py` calls `tokenizer.encode()` without `allowed_special=set()`; `chat.py` returns buffered JSON only (`stream=True` is not wired); `chat.py` expects caller-supplied `query_vector`. | Present these honestly as **known architectural gaps / Phase 2 roadmap items**—interviewers love engineers who recognize current boundaries! |

---

## 4. How to Use This Repository

1. **Follow [13-five-to-seven-day-study-plan.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/13-five-to-seven-day-study-plan.md)** as your daily driver (7-day full track for both SentinelRAG + KloudWorx, or 5-day intensive track).
2. **Never passively skim.** Every module ends with active-recall prompts or verification tasks. Speak explanations out loud before checking answers.
3. **Keep original `notes/` untouched** as an archive—everything you need for mastery and execution is self-contained inside `interview-preparation/`.

