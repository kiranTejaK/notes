# SentinelRAG & Backend Engineering Interview Hub

A clean, zero-duplication study repository built around the authoritative **SentinelRAG** AI Middleware Gateway guides, the high-density **Concept Revision** companion, and your **KloudWorx / Resume / Behavioral** interview defense.

---

## Repository Files (`7` Core Files)

| File | Scope | Role in Your Prep |
| :--- | :--- | :--- |
| **[01_SentinelRAG_Project_Walkthrough.md](file:///c:/Users/kiran/Desktop/notes/01_SentinelRAG_Project_Walkthrough.md)** | `2,679` lines | **Code Ground Truth:** Guided navigation, architecture, key implementation blocks, and line-by-line execution flow for the SentinelRAG codebase (`main.py`, `security.py`, `prompt_guard.py`, `embeddings.py`, `cache.py`, `search.py`, `router.py`, `chat.py`, SSE streaming). |
| **[02_SentinelRAG_Core_Concepts.md](file:///c:/Users/kiran/Desktop/notes/02_SentinelRAG_Core_Concepts.md)** | `3,150` lines | **CS & Systems Theory (Deep Study):** 68 prioritized concepts across ASGI/`uvloop`, Pydantic v2, `asyncio`, PostgreSQL 16 (`pgvector`/HNSW/GIN), Redis Lua/HNSW, distributed systems primitives, security, observability, testing, infra, and architectural trade-offs. |
| **[03_SentinelRAG_Interview_QA.md](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md)** | `5,809` lines | **205-Question Interview Bank:** Complete interview simulation (`Direct Answer -> Technical Depth -> SentinelRAG Connection -> Follow-ups`) spanning Project Pitch, Codebase Mechanics, Async/DB Internals, Scenarios, Trade-offs, Testing, and Staff CS Fundamentals. |
| **[04_SentinelRAG_Learning_Roadmap.md](file:///c:/Users/kiran/Desktop/notes/04_SentinelRAG_Learning_Roadmap.md)** | `287` lines | **Actionable Roadmap & Checklists:** 5-phase preparation framework with concrete completion criteria, common traps, and milestone tracking. |
| **[05_SentinelRAG_Concept_Revision.md](file:///c:/Users/kiran/Desktop/notes/05_SentinelRAG_Concept_Revision.md)** | `1,169` lines | **High-Density Revision Companion:** High-retention extraction of all 68 concepts: bulleted **Remember This** points, spoken **Explain It in One Minute** blockquotes (45–60s scripts), and embedded interview cross-examinations. |
| **[004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md)** | `469` lines | **Phase 2 (Resume & Behavioral):** `90s` opening pitch, KloudWorx defense (`Q1–Q23`: Slim $\to$ FastAPI, AWS SQS, S3 pre-signed URLs, MySQL ESR/deadlocks, Redis cache-aside, ES `nested`, Stripe HMAC, JWT/RBAC, Structlog, Pytest), `4` STAR scenarios, M.Sc. Chemistry pivot, and reverse questions. |
| **[resume/kiranteja-ai-engineer.pdf](file:///c:/Users/kiran/Desktop/notes/resume/kiranteja-ai-engineer.pdf)** | PDF | Official candidate resume. |

---

## Preparation Tracks & Time Allocation

Depending on your available interview preparation runway, choose between the **Normal Comprehensive Track (7 Days / ~21 Hours)** or the **Quick-Revision Track (3 Days / ~9 Hours)**.

```
                              [Choose Your Prep Track]
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
   [Normal Comprehensive Track]                      [Quick-Revision Track]
      7 Days (3.0 hrs/day)                            3 Days (3.0 hrs/day)
          Total: ~21 hrs                                  Total: ~9 hrs
                 │                                               │
  1. Deep Concepts (`02`)                         1. Rapid Recall & Spoken (`05`)
  2. IDE Walkthrough (`01`)                       2. Shortlisted Q&A (`03` 48 picks)
  3. Spoken Revision (`05`)                       3. Resume Defense (`004`)
  4. Full Q&A Practice (`03`)                     4. Whiteboard Architecture Flow
  5. Resume & Behavioral (`004`)
```

---

### Track A: Normal Comprehensive Track (7 Days · 3.0 hrs/day · ~21.0 hrs total)

Designed for thorough end-to-end mastery of systems architecture, internal code mechanics, and candidate interview defense.

- **Daily Cadence (3.0 hrs):**
  - `15m`: Prior-day active recall using [`05_SentinelRAG_Concept_Revision.md`](file:///c:/Users/kiran/Desktop/notes/05_SentinelRAG_Concept_Revision.md).
  - `90m`: In-depth study of code ground truth ([`01`](file:///c:/Users/kiran/Desktop/notes/01_SentinelRAG_Project_Walkthrough.md)) and CS theory ([`02`](file:///c:/Users/kiran/Desktop/notes/02_SentinelRAG_Core_Concepts.md)).
  - `45m`: Out-loud verbal question drills using [`03_SentinelRAG_Interview_QA.md`](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md) and [`004`](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md).
  - `30m`: Architecture diagramming, pipeline tracing, or checklist review.

| Day | Phase & Focus | Primary Target Sections | Time Allocation |
| :--- | :--- | :--- | :---: |
| **Day 1** *(Phase 1)* | **Gateway Fundamentals & Ingress:** ASGI vs WSGI, FastAPI `Depends`, Starlette middleware onion, config, security & rate limiting. | `01` (Secs 1–12), `02` (Concepts A1–A4, B1–B2), `03` (Q1–Q10, Q31–Q36) | **3.0 hrs** |
| **Day 2** *(Phase 1)* | **Vector Caching & Retrieval Pipeline:** Two-tier Redis cache, embeddings, prompt guard, `pgvector` HNSW, full-text search & RRF fusion. | `01` (Secs 13–21), `02` (Concepts A5–A10, A13–A15, A18), `03` (Q11–Q30, Q40–Q44) | **3.0 hrs** |
| **Day 3** *(Phase 1)* | **Resilience & Streaming Lifecycle:** Circuit breaker state machine, LiteLLM router, pre-first-chunk failover, SSE streaming, telemetry. | `01` (Secs 22–26), `02` (Concepts A16–A17, A20–A22, B11–B14), `03` (Q48–Q80) | **3.0 hrs** |
| **Day 4** *(Phase 1)* | **Production Scenarios & Trade-Offs:** Failure matrices, outage scenarios, capacity budgets, Redis clusters, and staff CS bank. | `02` (Concepts C1–C32), `03` (Selected drills from Q81–Q205), `05` (Categories A & B) | **3.0 hrs** |
| **Day 5** *(Phase 2)* | **KloudWorx Architecture & Cloud:** Slim $\to$ FastAPI refactor, AWS SQS decoupling, S3 pre-signed uploads, Stripe webhooks, JWT/RBAC. | `004resume-deepdive/` (Secs 1–2, Q1–Q7, Q18–Q20) | **3.0 hrs** |
| **Day 6** *(Phase 2)* | **Databases & Observability:** MySQL ESR index optimization, deadlocks, Redis cache-aside, Elasticsearch `nested`, Structlog & Pytest. | `004resume-deepdive/` (Q8–Q17, Q21–Q23), `02` (PostgreSQL/Redis deep dive) | **3.0 hrs** |
| **Day 7** *(Phase 2)* | **Mock Interview Simulation & Behavioral:** 90s pitch, 4 STAR behavioral scenarios, M.Sc. Chemistry career pivot, final checklists. | `004resume-deepdive/` (Secs 5–6), `05` (Spoken 1-min practice), `04` (Roadmap audit) | **3.0 hrs** |

---

### Track B: Quick-Revision Track (3 Days · 3.0 hrs/day · ~9.0 hrs total)

Designed for fast revision when an interview is scheduled in 48–72 hours. Focuses exclusively on spoken fluency, high-yield questions, and core behavioral defense.

- **Strategy:** Bypass the line-by-line code reading in `01` and heavy theory in `02`. Instead, use **[`05`](file:///c:/Users/kiran/Desktop/notes/05_SentinelRAG_Concept_Revision.md)** for rapid conceptual recall and drill the **48 High-Yield Questions** from **[`03`](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md)**.

| Day | Focus Area | High-Impact Revision Routine | Time Allocation |
| :--- | :--- | :--- | :---: |
| **Day 1** *(SentinelRAG Core)* | **Architecture Pitch, End-to-End Flow & Retrieval:** Master the 90s pitch, 5-stage pipeline, semantic cache, and hybrid RRF retrieval. | - Study & speak aloud **Category A (A1–A22)** in [`05_SentinelRAG_Concept_Revision.md`](file:///c:/Users/kiran/Desktop/notes/05_SentinelRAG_Concept_Revision.md) (`90m`).<br>- Drill **Shortlist Groups 1 & 2** below (17 questions: `Q1–Q10`, `Q13`, `Q15`, `Q22`, `Q25`, `Q27`, `Q28`, `Q43`, `Q49`, `Q65`) (`90m`). | **3.0 hrs** |
| **Day 2** *(Systems & Scenarios)* | **Resilience, Concurrency, Outages & Trade-Offs:** Circuit breaker, failover rules, PostgreSQL/Redis internals, fail-open vs closed. | - Review **Category B & C Takeaways** in [`05_SentinelRAG_Concept_Revision.md`](file:///c:/Users/kiran/Desktop/notes/05_SentinelRAG_Concept_Revision.md) (`60m`).<br>- Drill **Shortlist Groups 3, 4 & 5** below (31 questions: `Q31–Q42`, `Q54–Q70`, `Q82–Q124`) (`90m`).<br>- Practice whiteboard drawing of the 5-stage request pipeline (`30m`). | **3.0 hrs** |
| **Day 3** *(Resume & Behavioral)* | **KloudWorx Defense, STAR Stories & Final Polish:** Career pitch, distributed systems defense, behavioral scenarios, and reverse Qs. | - Drill KloudWorx technical defense in [`004resume-deepdive/`](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md) (`90m`).<br>- Rehearse 4 STAR stories and M.Sc. Chemistry pivot (`60m`).<br>- Rapidly verbalize 10 random 1-minute scripts from `05` (`30m`). | **3.0 hrs** |

---

## High-Yield Question Shortlist (`48` Realistic Picks from `205` in `03_SentinelRAG_Interview_QA.md`)

Instead of trying to memorize all `205` questions, focus your out-loud practice on these **48 realistic, non-redundant questions** that interviewers actually probe across Screening, Architecture, Code Deep-Dive, Systems Internals, and Production Debugging rounds (each linked directly to its exact line in [03_SentinelRAG_Interview_QA.md](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md)):

### 1. Project Pitch, Architecture & End-to-End Flow (`8` Picks)

- [Q1 (L36)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L36): What is SentinelRAG and what problem does it solve?
- [Q2 (L74)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L74): Walk through the high-level architecture.
- [Q5 (L153)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L153): How is multi-tenancy enforced end-to-end?
- [Q6 (L192)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L192): Explain the two-tier semantic caching strategy.
- [Q7 (L239)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L239): Why hybrid retrieval (`pgvector` + FTS + RRF) over dense-only?
- [Q8 (L288)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L288): How does SentinelRAG handle LLM provider failures?
- [Q9 (L337)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L337): End-to-end lifecycle of `POST /api/v1/chat/completions`.
- [Q10 (L3879)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3879) / [Q121 (L3795)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3795): Main scalability bottlenecks & 10x horizontal scaling plan.

### 2. Code Walkthrough & Implementation Mechanics (`9` Picks)

- [Q13 (L445)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L445) / [Q57 (L1774)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1774): `resolve_authorized_tenant()` & preventing tenant spoofing.
- [Q15 (L522)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L522) / [Q50 (L1635)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1635): `PythonScanBackend` (`SMEMBERS` + pipelined `HGETALL`) & its scaling ceiling ([Q72 (L2313)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2313)).
- [Q22 (L848)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L848) / [Q68 (L2170)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2170): Why `stream_complete()` refuses failover after `emitted_any_token`.
- [Q25 (L1001)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1001) / [Q78 (L4345)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4345): Lazy `vector_provider` callable avoiding wasted embedding calls on hash hits.
- [Q27 (L1055)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1055): `serialize_embedding()` IEEE-754 `struct.pack` + Base64 rationale.
- [Q28 (L3954)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3954) / [Q55 (L1734)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1734): Constant-time key check & `hmac.compare_digest()` timing-attack defense.
- [Q43 (L1484)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1484): Why `search_hybrid_candidates()` runs dense & sparse sequentially on one `AsyncSession`.
- [Q49 (L1592)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1592) / [Q52 (L4154)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4154): Why Redis Lua + `EVALSHA`/`NOSCRIPT` fallback over `GET/SET` or `MULTI/EXEC`.
- [Q65 (L2015)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2015): Why RRF `k=60` instead of weighted linear score normalization.

### 3. Core Tech: AsyncIO, FastAPI, PostgreSQL & Redis Internals (`12` Picks)

- [Q31 (L1106)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1106) & [Q32 (L1149)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1149): ASGI vs WSGI and single-worker event-loop multiplexing ([Q34 (L4033)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4033) CPU/blocking trap).
- [Q35 (L1245)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1245) & [Q135 (L4821)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4821): FastAPI `BackgroundTasks` vs `asyncio.create_task()` & unreferenced task leaks.
- [Q36 (L1290)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1290): `structlog` + `contextvars` vs `threading.local()`.
- [Q37 (L4049)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4049) & [Q130 (L4760)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4760): SSE client disconnect lifecycle & async backpressure.
- [Q41 (L1389)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1389) & [Q101 (L3311)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3311): `pgvector` HNSW (`M`, `ef_construction`) vs IVFFlat recall/RAM/rebuild trade-offs.
- [Q42 (L1433)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1433) & [Q44 (L4094)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4094): Persisted `tsvector` GIN index & `websearch_to_tsquery` vs `plainto_tsquery`.
- [Q161 (L5130)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5130) & [Q162 (L5144)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5144): PostgreSQL MVCC & `autovacuum` bloat on HNSW/GIN tables.
- [Q165 (L5175)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5175) & [Q171 (L5243)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5243): `PgBouncer` vs SQLAlchemy pool + `expire_on_commit=False` / `autoflush=False`.
- [Q178 (L5321)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5321) / [Q203 (L5577)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5577): Cache stampede & `HALF_OPEN` thundering-herd prevention.
- [Q179 (L5331)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5331): Redis eviction policy (`volatile-lru` vs `allkeys-lru` vs `noeviction`).

### 4. Design Decisions, Trade-Offs & Security (`9` Picks)

- [Q54 (L4192)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4192), [Q56 (L4206)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4206) & [Q103 (L4461)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4461): HMAC-SHA-256 + `API_KEY_PEPPER` vs JWT for API gateways.
- [Q59 (L4241)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4241) & [Q94 (L4447)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4447): Regex prompt-injection guard limitations & Unicode evasion.
- [Q63 (L1926)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1926) & [Q102 (L3357)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3357): Cosine distance threshold tuning (`0.08` vs `0.20`/`0.50`).
- [Q69 (L2221)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2221) & [Q97 (L3144)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3144): Tuning `failure_threshold=5`/`60s` & local vs Redis distributed circuit breaker.
- [Q70 (L2266)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2266): Graceful degradation when `EmbeddingGenerationException` triggers.
- [Q96 (L3105)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3105) & [Q74 (L2360)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2360): Python scan vs RediSearch `FT.SEARCH` HNSW & `TAG` pre-filtering.
- [Q98 (L3186)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3186) & [Q67 (L2118)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2118): Inline vs background cache write durability trade-off.
- [Q99 (L3229)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3229): Dense-only vs hybrid retrieval.
- [Q197 (L5516)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5516): Fail-open vs fail-closed matrix across dependencies.

### 5. Production Scenarios, Testing & Ops Debugging (`10` Picks)

- [Q82 (L2482)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2482): `/healthz` reports Redis `"down"` (`503`) while Redis container is healthy ([Q195 (L5496)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5496) k8s probes).
- [Q84 (L4387)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4387) & [Q85 (L2617)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2617): Cache hits never increment / zero hits after changing `EMBEDDING_MODEL` ([Q122 (L4632)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4632) 3072-dim migration).
- [Q86 (L2685)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2685) & [Q120 (L4618)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4618): Missing SSE `[DONE]`, worker restarts & Nginx `proxy_buffering off` ([Q156 (L5073)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5073), [Q198 (L5531)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5531)).
- [Q89 (L4419)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4419): Preventing unbounded per-tenant Redis cache growth.
- [Q90 (L2820)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2820): Malicious all-zero vector (`[0.0]*1536`) handling.
- [Q92 (L2965)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2965): Canary succeeds after 60s, yet traffic keeps hitting fallback OpenAI.
- [Q106 (L3448)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3448), [Q108 (L3506)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3506), [Q109 (L3565)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3565) & [Q114 (L3620)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3620): Testing FastAPI + SSE via `ASGITransport`, mocks & `dependency_overrides`.
- [Q118 (L4585)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4585) & [Q204 (L5587)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L5587): Production monitoring & Prometheus high-cardinality label traps.
- [Q124 (L4665)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4665): Compliance change to redact plaintext prompts from logs.
- [Q143 (L4924)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4924) & [Q148 (L4982)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4982): Docker non-root `UID 10001` & PID 1 `SIGTERM` shutdown.

> **What to Deprioritize on a Time Budget:** Skip or skim narrow code-trivia checks (`Q12`, `Q26`, `Q113`), duplicates (`Q187` repeats `Q179`), and `(Hypothetical)`-tagged deep SRE design questions (`Q189`, `Q190`, `Q201`, `Q202`, `Q205`) unless targeting a Staff SRE/Infra loop.

---

## Recommended 5-Step Learning & Revision Workflow

```
1. Deep Study:           02_SentinelRAG_Core_Concepts.md
                                │
2. Code Walkthrough:     01_SentinelRAG_Project_Walkthrough.md (with IDE)
                                │
3. Fast Recall & Drill:  05_SentinelRAG_Concept_Revision.md (Speak 1-minute scripts aloud)
                                │
4. Full Q&A Simulation:  03_SentinelRAG_Interview_QA.md (205 prioritized interview questions)
                                │
5. Progress Tracking:    04_SentinelRAG_Learning_Roadmap.md (5-phase milestone tracker)
```

---

## Cleanup Audit Summary

The repository was audited to remove obsolete, broken, or duplicate preparation files (still recoverable from git commit `2c34e3a` if ever needed):

- `001curriculum/`, `002phase-1-implementation/`, `003phase-2-interview-prep/` — Superseded by `01_SentinelRAG_Project_Walkthrough.md`, `02_SentinelRAG_Core_Concepts.md`, and `03_SentinelRAG_Interview_QA.md`.
- `interview-preparation/` — Generated earlier against the deleted `archive/supplementary.md` snapshot; superseded by `01`–`05` (which reflect the updated SentinelRAG implementation) and had broken links after `archive/` was removed.
- `New folder/` (`AI_BUILDER_TO_MASTER_BLUEPRINT.md`, `MASTER_STUDY_SCHEDULE.md`, `RESUME_AND_INTERVIEW_GUIDE.md`) — Obsolete drafts superseded by the kept files above.
