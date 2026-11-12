# SentinelRAG & Backend Engineering Interview Hub

A clean, zero-duplication study repository built around the three authoritative **SentinelRAG** guides plus your **KloudWorx / Resume / Behavioral** defense.

---

## Kept Repository Files (`6` Core Files)

| File | Scope | Role in Your Prep |
| :--- | :--- | :--- |
| **[01_SentinelRAG_Project_Walkthrough.md](file:///c:/Users/kiran/Desktop/notes/01_SentinelRAG_Project_Walkthrough.md)** | `2,679` lines | **Code Ground Truth:** Guided navigation, key blocks, and line-by-line execution flow for the current SentinelRAG codebase (`main.py`, `security.py`, `prompt_guard.py`, `embeddings.py`, `cache.py`, `search.py`, `router.py`, `chat.py`, SSE streaming). |
| **[02_SentinelRAG_Core_Concepts.md](file:///c:/Users/kiran/Desktop/notes/02_SentinelRAG_Core_Concepts.md)** | `2,023` lines | **CS & Systems Theory:** ASGI/`uvloop`, Pydantic v2, `asyncio`, PostgreSQL 16 (`pgvector`/HNSW/GIN), Redis Lua/HNSW, distributed systems primitives, security, observability, testing, infra, invariants cheat-sheet, and trade-offs matrix. |
| **[03_SentinelRAG_Interview_QA.md](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md)** | `4,680` lines | **205-Question Interview Bank:** Structured spoken answers (`Direct Answer -> Technical Depth -> SentinelRAG Connection -> Follow-up`) spanning Intro, Code, Tech, Why/What-If, Scenarios, Trade-offs, Testing, Prod Ops, and Part 9 CS/Systems. |
| **[004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md)** | `469` lines | **Phase 2 (Resume & Behavioral):** `90s` opening pitch, KloudWorx defense (`Q1–Q23`: Slim $\to$ FastAPI, AWS SQS, S3 pre-signed URLs, MySQL ESR/deadlocks, Redis cache-aside, ES `nested`, Stripe HMAC webhooks, JWT/RBAC, Structlog, Pytest), `4` STAR scenarios, M.Sc. Chemistry pivot, and interviewer questions. *(Use `01`–`03` above instead of Section 4 here for SentinelRAG specifics.)* |
| **[resume/kiranteja-ai-engineer.pdf](file:///c:/Users/kiran/Desktop/notes/resume/kiranteja-ai-engineer.pdf)** | PDF | Official candidate resume. |

---

## Study Schedule & Estimated Time Allocations

Standard pace is **`~3.0 hrs/day` over 7 days (`~21 hrs` total)**, or **`~3.0–3.5 hrs/day` over 5 days (`~16.5 hrs` total)**:
- **Daily Cadence (`3.0 hrs`):** `15m` prior-day recall $\to$ `90m` focused reading (`01` / `02` / `004`) $\to$ `45m` out-loud Q&A (`03` / `004`) $\to$ `30m` diagramming or checklist drill.

| Day | Phase & Focus | Target Sections | Est. Time |
| :--- | :--- | :--- | :---: |
| **Day 1** *(Phase 1)* | SentinelRAG Gateway, ASGI, Config, DB, Redis, Security & Rate Limiter | `01` (Secs 1–12), `02` (Secs 1–5), `03` (Part 1 Q1–Q10) | **3.0 hrs** |
| **Day 2** *(Phase 1)* | Embeddings, Prompt Guard, Semantic Cache, Hybrid Search & RRF | `01` (Secs 13–21), `02` (Secs 6–8), `03` (Part 2 Q11–Q30) | **3.0 hrs** |
| **Day 3** *(Phase 1)* | Circuit Breaker, Router, `chat.py` E2E, Telemetry, Testing & Infra | `01` (Secs 22–26), `02` (Secs 9–12), `03` (Parts 3–4 Q31–Q80) | **3.0 hrs** |
| **Day 4** *(Phase 1)* | Scenarios, Trade-Offs, Ops, Staff Invariants & Core CS Bank | `02` (Secs 13–16), `03` (Parts 5–9 Q81–Q205 selective drills) | **3.0 hrs** |
| **Day 5** *(Phase 2)* | KloudWorx Distributed Systems (Slim/FastAPI, SQS, S3, Stripe, RBAC) | `004resume-deepdive/` (`Sec 1–2`, `Q1–Q7`, `Q18–Q20`) | **3.0 hrs** |
| **Day 6** *(Phase 2)* | MySQL ESR/Deadlocks, Redis Cache-Aside, ES `nested`, Structlog & Pytest | `004resume-deepdive/` (`Q8–Q17`, `Q21–Q23`) + `02` (Appendix) | **3.0 hrs** |
| **Day 7** *(Phase 2)* | Behavioral STAR, M.Sc. Chemistry Pivot, Timed Mock & Final Checklists | `004resume-deepdive/` (`Sec 5–6`), `01` (`Sec 26`), `03` (Checklist) | **3.0 hrs** |

> **Compressed 5-Day Track (`~16.5 hrs`):** Day 1 (`3.0h`: `01` Secs 1–15 + `02` Secs 1–5) $\to$ Day 2 (`3.5h`: `01` Secs 16–26 + `02` Secs 6–12) $\to$ Day 3 (`3.5h`: `02` Secs 13–16 + `03` Parts 1–8) $\to$ Day 4 (`3.5h`: `004resume-deepdive/` `Q1–Q23` + `03` Part 9) $\to$ Day 5 (`3.0h`: STAR stories, Chemistry pivot, mock Q&A & final checklists).

---

## High-Yield Question Shortlist (`48` Realistic Picks from `205` in `03_SentinelRAG_Interview_QA.md`)

Instead of trying to memorize all `205` questions, focus your out-loud practice on these **48 realistic, non-redundant questions** that interviewers actually probe across Screening, Architecture, Code Deep-Dive, Systems Internals, and Production Debugging rounds (each linked directly to its line in [03_SentinelRAG_Interview_QA.md](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md)):

### 1. Project Pitch, Architecture & End-to-End Flow (`8` Picks)
- [Q1 (L30)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L30): What is SentinelRAG and what problem does it solve?
- [Q2 (L54)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L54): Walk through the high-level architecture.
- [Q5 (L127)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L127): How is multi-tenancy enforced end-to-end?
- [Q6 (L150)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L150): Explain the two-tier semantic caching strategy.
- [Q7 (L172)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L172): Why hybrid retrieval (`pgvector` + FTS + RRF) over dense-only?
- [Q8 (L196)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L196): How does SentinelRAG handle LLM provider failures?
- [Q9 (L220)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L220): End-to-end lifecycle of `POST /api/v1/chat/completions`.
- [Q10 (L250)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L250) / [Q121 (L2821)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2821): Main scalability bottlenecks & 10x horizontal scaling plan.

### 2. Code Walkthrough & Implementation Mechanics (`9` Picks)
- [Q13 (L325)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L325) / [Q57 (L1324)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1324): `resolve_authorized_tenant()` & preventing tenant spoofing.
- [Q15 (L369)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L369) / [Q50 (L1167)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1167): `PythonScanBackend` (`SMEMBERS` + pipelined `HGETALL`) & its scaling ceiling ([Q72](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1666)).
- [Q22 (L526)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L526) / [Q68 (L1577)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1577): Why `stream_complete()` refuses failover after `emitted_any_token`.
- [Q25 (L590)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L590) / [Q78 (L1786)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1786): Lazy `vector_provider` callable avoiding wasted embedding calls on hash hits.
- [Q27 (L637)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L637): `serialize_embedding()` IEEE-754 `struct.pack` + Base64 rationale.
- [Q28 (L660)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L660) / [Q55 (L1283)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1283): Constant-time key check & `hmac.compare_digest()` timing-attack defense.
- [Q43 (L1008)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1008): Why `search_hybrid_candidates()` runs dense & sparse sequentially on one `AsyncSession`.
- [Q49 (L1142)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1142) / [Q52 (L1210)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1210): Why Redis Lua + `EVALSHA`/`NOSCRIPT` fallback over `GET/SET` or `MULTI/EXEC`.
- [Q65 (L1508)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1508): Why RRF `k=60` instead of weighted linear score normalization.

### 3. Core Tech: AsyncIO, FastAPI, PostgreSQL & Redis Internals (`12` Picks)
- [Q31 (L726)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L726) & [Q32 (L751)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L751): ASGI vs WSGI and single-worker event-loop multiplexing ([Q34](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L792) CPU/blocking trap).
- [Q35 (L815)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L815) & [Q135 (L3159)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3159): FastAPI `BackgroundTasks` vs `asyncio.create_task()` & unreferenced task leaks.
- [Q36 (L837)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L837): `structlog` + `contextvars` vs `threading.local()`.
- [Q37 (L860)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L860) & [Q130 (L3052)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3052): SSE client disconnect lifecycle & async backpressure.
- [Q41 (L960)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L960) & [Q101 (L2338)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2338): `pgvector` HNSW (`M`, `ef_construction`) vs IVFFlat recall/RAM/rebuild trade-offs.
- [Q42 (L983)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L983) & [Q44 (L1027)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1027): Persisted `tsvector` GIN index & `websearch_to_tsquery` vs `plainto_tsquery`.
- [Q161 (L3735)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3735) & [Q162 (L3756)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3756): PostgreSQL MVCC & `autovacuum` bloat on HNSW/GIN tables.
- [Q165 (L3816)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3816) & [Q171 (L3942)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3942): `PgBouncer` vs SQLAlchemy pool + `expire_on_commit=False` / `autoflush=False`.
- [Q178 (L4089)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4089) / [Q203 (L4617)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4617): Cache stampede & `HALF_OPEN` thundering-herd prevention.
- [Q179 (L4109)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4109): Redis eviction policy (`volatile-lru` vs `allkeys-lru` vs `noeviction`).

### 4. Design Decisions, Trade-Offs & Security (`9` Picks)
- [Q54 (L1260)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1260), [Q56 (L1305)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1305) & [Q103 (L2379)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2379): HMAC-SHA-256 + `API_KEY_PEPPER` vs JWT for API gateways.
- [Q59 (L1372)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1372) & [Q94 (L2164)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2164): Regex prompt-injection guard limitations & Unicode evasion.
- [Q63 (L1465)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1465) & [Q102 (L2358)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2358): Cosine distance threshold tuning (`0.08` vs `0.20`/`0.50`).
- [Q69 (L1596)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1596) & [Q97 (L2242)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2242): Tuning `failure_threshold=5`/`60s` & local vs Redis distributed circuit breaker.
- [Q70 (L1618)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1618): Graceful degradation when `EmbeddingGenerationException` triggers.
- [Q96 (L2215)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2215) & [Q74 (L1709)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1709): Python scan vs RediSearch `FT.SEARCH` HNSW & `TAG` pre-filtering.
- [Q98 (L2266)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2266) & [Q67 (L1557)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1557): Inline vs background cache write durability trade-off.
- [Q99 (L2289)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2289): Dense-only vs hybrid retrieval.
- [Q197 (L4487)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4487): Fail-open vs fail-closed matrix across dependencies.

### 5. Production Scenarios, Testing & Ops Debugging (`10` Picks)
- [Q82 (L1879)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1879): `/healthz` reports Redis `"down"` (`503`) while Redis container is healthy ([Q195](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4443) k8s probes).
- [Q84 (L1930)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1930) & [Q85 (L1953)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1953): Cache hits never increment / zero hits after changing `EMBEDDING_MODEL` ([Q122](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2850) 3072-dim migration).
- [Q86 (L1976)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L1976) & [Q120 (L2798)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2798): Missing SSE `[DONE]`, worker restarts & Nginx `proxy_buffering off` ([Q156](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3629), [Q198](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4512)).
- [Q89 (L2044)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2044): Preventing unbounded per-tenant Redis cache growth.
- [Q90 (L2071)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2071): Malicious all-zero vector (`[0.0]*1536`) handling.
- [Q92 (L2118)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2118): Canary succeeds after 60s, yet traffic keeps hitting fallback OpenAI.
- [Q106 (L2451)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2451), [Q108 (L2489)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2489), [Q109 (L2514)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2514) & [Q114 (L2650)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2650): Testing FastAPI + SSE via `ASGITransport`, mocks & `dependency_overrides`.
- [Q118 (L2748)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2748) & [Q204 (L4637)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L4637): Production monitoring & Prometheus high-cardinality label traps.
- [Q124 (L2903)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L2903): Compliance change to redact plaintext prompts from logs.
- [Q143 (L3351)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3351) & [Q148 (L3463)](file:///c:/Users/kiran/Desktop/notes/03_SentinelRAG_Interview_QA.md#L3463): Docker non-root `UID 10001` & PID 1 `SIGTERM` shutdown.

> **What to Deprioritize on a Time Budget:** Skip or skim narrow code-trivia checks (`Q12`, `Q26`, `Q113`), duplicates (`Q187` repeats `Q179`), and `(Hypothetical)`-tagged deep SRE design questions (`Q189`, `Q190`, `Q201`, `Q202`, `Q205`) unless targeting a Staff SRE/Infra loop.

---

## Cleanup Audit Summary

Removed obsolete/superseded directories (still recoverable from git commit `2c34e3a` if ever needed):
- `001curriculum/`, `002phase-1-implementation/`, `003phase-2-interview-prep/` — Superseded by `01_SentinelRAG_Project_Walkthrough.md`, `02_SentinelRAG_Core_Concepts.md`, and `03_SentinelRAG_Interview_QA.md`.
- `interview-preparation/` — Generated earlier against the deleted `archive/supplementary.md` snapshot; superseded by `01`–`03` (which reflect the updated SentinelRAG implementation) and had broken links after `archive/` was removed.
- `New folder/` (`AI_BUILDER_TO_MASTER_BLUEPRINT.md`, `MASTER_STUDY_SCHEDULE.md`, `RESUME_AND_INTERVIEW_GUIDE.md`) — Previously staged obsolete drafts superseded by the kept files above.
