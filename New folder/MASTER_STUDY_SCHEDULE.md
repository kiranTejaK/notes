# 🎯 Master Study Schedule: Two-Phase Career & Interview Blueprint

> **Target Identity:** Senior Backend Engineer specializing in Distributed Systems, High-Throughput APIs & Applied AI/RAG Infrastructure.  
> **Star Project:** SentinelRAG (Enterprise AI Gateway & Hybrid Retrieval Engine)  
> **Production Background:** ~3 Years Backend Engineering at KloudWorx (Python, FastAPI, AWS SQS, S3, MySQL, Redis)  
> **Workspace:** `c:\Users\kiran\Desktop\notes`

---

## 🗺️ Master Plan at a Glance

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│               PHASE 1: MASTERING SENTINELRAG IN AND OUT (DAYS 1 – 4)                   │
│               The Star Project & AI Backend Platform Engineering Anchor                │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
  ├── Day 1: Topology, ASGI & Token Bucket Rate Limiting (Redis Lua)
  ├── Day 2: PostgreSQL 16 Single-Store: pgvector HNSW + BM25 Lexical
  ├── Day 3: Reciprocal Rank Fusion (RRF k=60) & Redis Two-Tier Semantic Caching (<=0.08)
  └── Day 4: Resilient Multi-Provider Routing (Circuit Breakers) & Socket-to-LLM Trace
                                            │
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│               PHASE 2: OVERALL INTERVIEW READINESS & DEFENSE (DAYS 5 – 7)              │
│               Production Experience, Systems Scrutiny, Behavioral & EM Loops           │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
  ├── Day 5: KloudWorx Architecture: AWS SQS, S3 Pre-Signed URLs & Stripe Idempotency
  ├── Day 6: MySQL B+Tree ESR Indexing, Redis Cache-Aside & Python AsyncIO Internals
  └── Day 7: Behavioral Masterclass (STAR Scenarios), Chemistry Pivot & Whiteboard Mock
```

---

## 🗂️ Complete Repository File Audit (Active vs. Archive)

Before diving into study sessions, use this audit to eliminate cognitive clutter:

| Directory / File | Status | Role in Your Preparation |
| :--- | :---: | :--- |
| **`004resume-deepdive/`** | 🔥 **ACTIVE (Tier 1)** | **Top Priority:** Complete defense of your resume, KloudWorx work, and STAR scenarios. |
| **`003phase-2-interview-prep/`** | 🔥 **ACTIVE (Tier 1)** | **Top Priority:** 56 Technical Q&As, trade-offs, system design defenses, and whiteboard drill. |
| **`002phase-1-implementation/`** | 🔥 **ACTIVE (Tier 1)** | **Top Priority:** SentinelRAG core topology, mathematical concepts, and 24-step trace. |
| **`MASTER_STUDY_SCHEDULE.md`** | 🔥 **ACTIVE (Tier 1)** | **Top Priority:** Your daily operational roadmap and progress tracker. |
| **`001curriculum/`** | 📖 **REFERENCE (Tier 2)** | **On-Demand:** Deep code-level walk-throughs (Day 1 to Day 7); consult only if stuck. |
| **`resume/kiranteja-ai-engineer.pdf`** | 📖 **REFERENCE (Tier 2)** | Your official source resume. |
| **`interview_mastery/`** | 📦 **ARCHIVE (Tier 3)** | **Redundant:** Older monolithic draft superseded by `002` and `003`. Skip. |
| **`archive/daily_logs/`** | 📦 **ARCHIVE (Tier 3)** | **Historical:** Raw chronological dev logs from project generation. Skip. |
| **`archive/supplementary.md`** | 📦 **ARCHIVE (Tier 3)** | **Historical:** 336 KB raw data dump. Skip. |
| **`RESUME_AND_INTERVIEW_GUIDE.md`**| 📦 **ARCHIVE (Tier 3)** | **Superseded:** Early draft superseded by `004resume-deepdive/`. Skip. |

---

## 🛡️ PHASE 1: Mastering SentinelRAG In and Out (Days 1–4)

*Goal: Speak with instinctive, author-level authority on every algorithm, database query, data structure, and failure mode of SentinelRAG.*

---

### 📅 Day 1: Topology, ASGI Core & Distributed Rate Limiting (Redis Lua)

#### 1. Core Learning Objectives
- Master the **30-second and 2-minute elevator pitches** for SentinelRAG.
- Understand the **ASGI Request Lifecycle** (`scope`, `receive`, `send`) and how FastAPI/Uvicorn schedules coroutines.
- Master the **Token Bucket Algorithm** implemented in Redis via an atomic Lua script.

#### 2. Mandatory Reading
- [002phase-1-implementation/01-project-overview.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/01-project-overview.md) (Elevator Pitches & Problem Space)
- [002phase-1-implementation/02-architecture-and-contracts.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/02-architecture-and-contracts.md) (Topology & Boundaries)
- [002phase-1-implementation/03-core-concepts.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/03-core-concepts.md) (Section 1: Token Bucket in Lua)
- [003phase-2-interview-prep/07-technology-qa-bank.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/07-technology-qa-bank.md) (Questions 1–7: FastAPI, ASGI, Uvicorn)

#### 3. High-Stakes Interview Defense Drills
- **Q:** *"Why implement rate limiting in a Redis Lua script instead of standard Redis commands (`INCR`, `EXPIRE`) or Python middleware memory?"*
  - **Defense:** In-memory middleware fails in multi-instance deployments behind a load balancer because bucket state is fragmented. Standard Redis commands (`GET` then `SET`) introduce race conditions under concurrency (time-of-check to time-of-use bug). Multi-command transactions (`MULTI/EXEC`) cannot do conditional refills based on time elapsed. An atomic Lua script executes in a single Redis thread with zero network round trips and zero race conditions.
- **Q:** *"What happens if a developer runs a blocking call like `requests.get()` inside an `async def` endpoint?"*
  - **Defense:** Python's event loop runs on a single thread. Any synchronous blocking I/O freezes the event loop for all concurrent client connections. In SentinelRAG, all HTTP calls use non-blocking `httpx.AsyncClient` and database queries use `asyncpg`.

---

### 📅 Day 2: Single-Store PostgreSQL 16 (pgvector HNSW & Lexical BM25)

#### 1. Core Learning Objectives
- Defend the **Single-Store Architecture** (PostgreSQL 16) vs dual-store database sprawl (Pinecone / Weaviate + Postgres).
- Understand **HNSW (Hierarchical Navigable Small World)** graphs vs IVFFlat indexing.
- Master **PostgreSQL Lexical Search** using `tsvector`, `tsquery`, and GIN indexes.

#### 2. Mandatory Reading
- [002phase-1-implementation/03-core-concepts.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/03-core-concepts.md) (Section 3: pgvector HNSW & Section 4: Lexical BM25)
- [003phase-2-interview-prep/05-tradeoffs-and-system-design.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/05-tradeoffs-and-system-design.md) (Trade-off 1: Single-Store vs Dedicated Vector DB)
- [003phase-2-interview-prep/07-technology-qa-bank.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/07-technology-qa-bank.md) (Questions 13–18: pgvector & SQL)

#### 3. High-Stakes Interview Defense Drills
- **Q:** *"Why not use Pinecone or Qdrant? Isn't a dedicated vector database faster?"*
  - **Defense:** Pinecone introduces the **Dual-Write Consistency Trap**: when a user updates or deletes a document, you must coordinate updates across PostgreSQL (metadata) and Pinecone (vectors). Network partitions or worker crashes cause the two systems to drift. With PostgreSQL 16 + `pgvector`, metadata, relational columns, JSONB attributes, and 1536-dimensional embeddings live in the exact same table row within a single ACID transaction. Furthermore, HNSW indexing (`m=16, ef_construction=64`) delivers sub-15ms vector retrieval.
- **Q:** *"What is the difference between HNSW and IVFFlat in pgvector?"*
  - **Defense:** IVFFlat partitions vector space into Voronoi cells via k-means clustering. It requires existing data to train centroid clusters, has lower recall on dynamic data, and degrades as new vectors are inserted. HNSW builds a multi-layer graph of skip-lists. It requires no training phase, maintains $>95\%$ recall even with continuous inserts, and offers $O(\log N)$ search time.

---

### 📅 Day 3: Reciprocal Rank Fusion (RRF) & Two-Tier Semantic Caching

#### 1. Core Learning Objectives
- Master the mathematics of **Reciprocal Rank Fusion ($k=60$)** and explain why simple linear score weighting fails.
- Understand the **Two-Tier Cache Hierarchy**: Stage 1 Exact MD5 Hash vs Stage 2 Redis Vector Semantic Search.
- Explain IEEE 754 binary float packing (`struct.pack('<1536f')`) and cosine distance thresholds ($\le 0.08$).

#### 2. Mandatory Reading
- [002phase-1-implementation/03-core-concepts.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/03-core-concepts.md) (Section 2: Float Packing, Section 5: RRF, Section 6: Semantic Cache)
- [003phase-2-interview-prep/05-tradeoffs-and-system-design.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/05-tradeoffs-and-system-design.md) (Trade-off 2: Reciprocal Rank Fusion vs Linear Weighted Fusion)
- [003phase-2-interview-prep/07-technology-qa-bank.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/07-technology-qa-bank.md) (Questions 8–12: Redis & aioredis)

#### 3. High-Stakes Interview Defense Drills
- **Q:** *"Why use RRF instead of normalizing vector scores and BM25 scores and doing `0.7 * vector + 0.3 * BM25`?"*
  - **Defense:** Cosine distance is strictly bounded $[0.0, 2.0]$, whereas BM25 scores are unbounded positive numbers $[0.0, \infty)$ dependent on document frequency and query term density. Standardizing or min-max scaling BM25 requires knowing the global score distribution of the entire dataset. RRF bypasses score calibration entirely by operating purely on **ordinal rank positions**:
    $$RRF\_Score(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
    Setting $k=60$ prevents top-ranked outliers from dominating and balances dense semantic recall with exact keyword matches.
- **Q:** *"Why is the semantic cache distance threshold set to $\le 0.08$?"*
  - **Defense:** In a 1536-dimensional normalized cosine space, distances between $0.00$ and $0.08$ represent semantic paraphrases with identical factual intent (e.g., *"What is our parental leave policy?"* vs *"Explain the maternity and paternity leave rules"*). Thresholds above $0.12$ introduce false-positive cache hits, serving incorrect context. Setting it at $\le 0.08$ guarantees zero hallucinated hits while offloading 40–60% of repetitive enterprise LLM calls in $<20\text{ms}$.

---

### 📅 Day 4: Resilient Multi-Provider Routing & Socket-to-LLM Trace

#### 1. Core Learning Objectives
- Master the **Michael Nygard Circuit Breaker** finite state machine (`CLOSED`, `OPEN`, `HALF_OPEN`).
- Trace an incoming HTTP request across all **24 sequential steps** from client socket to LLM provider and back.
- Defend retry strategies: `Tenacity` exponential backoff with decorrelated full jitter.

#### 2. Mandatory Reading
- [002phase-1-implementation/03-core-concepts.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/03-core-concepts.md) (Section 7: Circuit Breaker State Machine)
- [002phase-1-implementation/04-codebase-and-trace.md](file:///c:/Users/kiran/Desktop/notes/002phase-1-implementation/04-codebase-and-trace.md) (The 24-Step Request Trace)
- [003phase-2-interview-prep/05-tradeoffs-and-system-design.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/05-tradeoffs-and-system-design.md) (Trade-off 3: Sliding-Window Circuit Breakers vs Naive Retries)
- [003phase-2-interview-prep/08-whiteboard-and-revision.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/08-whiteboard-and-revision.md) (5-Minute Whiteboard Script)

#### 3. High-Stakes Interview Defense Drills
- **Q:** *"Why not just wrap your LLM calls in a simple `try/except` with 3 retries?"*
  - **Defense:** Naive retries under provider downtime create a **Retry Storm (Thundering Herd)** that amplifies the outage. If Groq is returning HTTP 429 or 503, retrying immediately from 500 concurrent client requests floods the failing provider and wastes server worker threads. A Circuit Breaker fails fast in $<5\text{ms}$ once the threshold (e.g., 5 consecutive failures) is reached, immediately routing traffic to the fallback model (OpenAI `gpt-4o-mini`) without hitting the broken upstream service.

---

## 💼 PHASE 2: Overall Interview Readiness (Days 5–7)

*Goal: Defend your nearly 3 years of production engineering at KloudWorx, your core backend acumen, and your behavioral/leadership stories.*

---

### 📅 Day 5: KloudWorx Architecture (AWS SQS, S3 Pre-Signed URLs & Stripe)

#### 1. Core Learning Objectives
- Defend **Event-Driven Document Processing** using AWS SQS, handling duplicate messages, and dead-letter queues (DLQ).
- Explain **Direct S3 File Uploads via Pre-Signed URLs** and security mechanisms (magic bytes vs file extensions).
- Master **Stripe Webhook Verification**, HMAC-SHA256 signatures, and database idempotency.

#### 2. Mandatory Reading
- [004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md) (Sections 1, 2, Q4–Q7, and Q18–Q20)

#### 3. High-Stakes Interview Defense Drills
- **Q:** *"SQS provides at-least-once delivery. How did your worker ensure documents weren't processed twice?"*
  - **Defense:** Every SQS message carried an `idempotency_key` (UUIDv4). When a worker picked up a task, it acquired a distributed Redis lock (`SET lock:{idempotency_key} NX EX 300`) and checked the `document_audit_status` table in MySQL. If the status was already `PROCESSED` or `IN_PROGRESS`, the worker immediately acknowledged and deleted the message from SQS without running expensive OCR/watermarking tasks.
- **Q:** *"Why didn't you upload documents directly through your FastAPI server?"*
  - **Defense:** Direct uploads stream file bytes through the backend application server. If 500 users concurrently upload 20MB onboarding PDFs, backend memory spikes by 10GB and ties up worker connections. By generating an S3 pre-signed URL (`boto3.generate_presigned_url`), clients upload directly to S3. To prevent spoofing, our downstream SQS worker validates binary magic bytes (`%PDF-`) from byte range `0-7` rather than trusting the client-supplied MIME type.

---

### 📅 Day 6: Database Optimization (B+Tree ESR Rule), Redis Caching & Python Internals

#### 1. Core Learning Objectives
- Defend **MySQL Query Tuning**: B+Tree mechanics, the ESR (Equality, Sort, Range) rule, and stored procedure trade-offs.
- Master the **Cache-Aside Pattern**, cache stampede (thundering herd) mitigations, and why writes must `DEL` rather than update cache keys.
- Master **Python Internals**: Global Interpreter Lock (GIL), Generational Garbage Collection, and AsyncIO event loops.

#### 2. Mandatory Reading
- [004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md) (Q8–Q14 & Technical Q&A Bank)
- [003phase-2-interview-prep/07-technology-qa-bank.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/07-technology-qa-bank.md) (Questions 48–56: Pydantic v2, Pytest, Docker)

#### 3. High-Stakes Interview Defense Drills
- **Q:** *"Explain the ESR rule in database indexing and why column order matters."*
  - **Defense:** InnoDB indexes are structured as B+Trees. To minimize disk reads:
    - **E (Equality):** Exact matches (`WHERE tenant_id = 42`) come first, immediately narrowing down the search tree.
    - **S (Sort):** The column used in `ORDER BY` (`ORDER BY created_at DESC`) comes next. Because the index is physically sorted on disk, MySQL reads leaf nodes in order, completely avoiding a costly in-memory or on-disk `filesort`.
    - **R (Range):** Inequalities (`AND updated_at >= ...`) come last, because once a B-Tree encounters a range scan, subsequent indexed columns cannot be used for sorting or direct point lookups.
- **Q:** *"Why invalidate Redis with `DEL` instead of updating the value during database writes?"*
  - **Defense:** Updating the cache on write creates a race condition between concurrent updates. If Update 2 commits to MySQL after Update 1, but Update 1's network packet reaches Redis after Update 2, the cache permanently stores stale data. Deleting the key (`DEL`) inside the database transaction ensures the next read lazily repopulates the single source of truth from the database.

---

### 📅 Day 7: Behavioral Masterclass (STAR), Career Pivot & Whiteboard Simulation

#### 1. Core Learning Objectives
- Deliver the **90-Second Opening Pitch** flawlessly without stumbling.
- Own the **M.Sc. Chemistry to Software Engineering** pivot story as a unique analytical advantage.
- Master the **4 Core Behavioral Scenarios** (Outage, Conflict, Leadership, Technical Debt).
- Execute a **5-Minute Whiteboard Drawing** of SentinelRAG from memory.

#### 2. Mandatory Reading
- [004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md](file:///c:/Users/kiran/Desktop/notes/004resume-deepdive/RESUME_DEEP_DIVE_AND_INTERVIEW_DEFENSE.md) (Sections 2, 5, and 6)
- [003phase-2-interview-prep/08-whiteboard-and-revision.md](file:///c:/Users/kiran/Desktop/notes/003phase-2-interview-prep/08-whiteboard-and-revision.md) (Full Whiteboard Script)

#### 3. High-Stakes Behavioral Defense Drills
- **Q:** *"Why did you switch from Chemistry to Software Engineering?"*
  - **Defense:** *"My Master's in Chemistry was grounded in scientific rigor: hypothesis testing, reaction kinetics, tracking variables, and understanding how microscopic changes affect macroscopic systems. When I discovered programming, I realized backend engineering is the exact same discipline applied to data: tracking state transitions, analyzing concurrency kinetics, and debugging root causes with data rather than intuition. I spent nearly 3 years at KloudWorx building enterprise platforms, and that scientific training makes me relentless when debugging complex distributed bugs."*
- **Q:** *"Tell me about a time you resolved a major production outage."*
  - **Defense:** Walk through **Scenario 1** in the guide: MySQL CPU at 98% during quarter-end onboarding rush, slow query log identifying missing composite index on audit logs, spinning up read replica, applying online zero-lock schema migration, and adding automated CI query plan gating.

---

## 📊 Daily Study Allocation Matrix

| Phase | Days | Daily Focus Area | Primary Target File | Time Share |
| :---: | :---: | :--- | :--- | :---: |
| **Phase 1** | **Day 1** | SentinelRAG Topology, ASGI & Redis Lua Rate Limiting | `002phase-1-implementation/01-03` | 3 Hours |
| **Phase 1** | **Day 2** | PostgreSQL 16 Single-Store, pgvector HNSW & BM25 | `003phase-2-interview-prep/05` | 3 Hours |
| **Phase 1** | **Day 3** | Reciprocal Rank Fusion ($k=60$) & Redis Semantic Cache | `002phase-1-implementation/03` | 3 Hours |
| **Phase 1** | **Day 4** | Circuit Breakers, Multi-Model Routing & 24-Step Trace | `002phase-1-implementation/04` | 3 Hours |
| **Phase 2** | **Day 5** | KloudWorx SQS, S3 Pre-Signed URLs & Stripe Webhooks | `004resume-deepdive/` | 3 Hours |
| **Phase 2** | **Day 6** | MySQL B+Tree ESR Indexing, Redis Cache-Aside & Python | `004resume-deepdive/` & `003/07` | 3 Hours |
| **Phase 2** | **Day 7** | Behavioral STAR Scenarios, Chemistry Pivot & Whiteboard | `004resume-deepdive/` & `003/08` | 3 Hours |

---

## 🏁 Final Readiness Verification

Before heading into your interview loop, verify you can answer these 5 questions out loud without looking at notes:
1. [ ] Can you draw the SentinelRAG 5-box topology on paper in 90 seconds?
2. [ ] Can you explain why RRF ($k=60$) beats linear score normalization?
3. [ ] Can you explain the ESR rule for composite database indexes?
4. [ ] Can you explain how you ensured idempotency in AWS SQS worker consumers?
5. [ ] Can you deliver your 90-second "Tell me about yourself" pitch smoothly and assertively?
