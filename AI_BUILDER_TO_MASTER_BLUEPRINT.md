# The AI-Builder to Senior Engineer Battle Plan
## 7-Session Mastery Program for SentinelRAG Technical Defense

> **The Reality:** You built this enterprise AI gateway using an AI agent. The architecture, algorithms, and code are solid—now you must close the **credibility gap** between *having working code in your repo* and *demonstrating instinctive, senior-level engineering ownership* in interviews.

---

## 🗺️ Master Curriculum at a Glance

| Session | Core Topic | Production Engine | Tactical Scar / Drill | Defensible Interview Claim |
| :---: | :--- | :--- | :--- | :--- |
| **1** | **ASGI Architecture & Concurrency** | FastAPI, Uvicorn, Structlog | Trigger & fix event loop starvation | Single-threaded async I/O vs thread-per-request scaling |
| **2** | **PostgreSQL 16 & pgvector Storage** | SQLAlchemy Async, asyncpg, HNSW | Inspect HNSW graph & memory settings | Single-store ACID vs Pinecone/Postgres dual-write sprawl |
| **3** | **Hybrid Retrieval & RRF Fusion** | Dense Vector + Sparse BM25 | Run RRF calculation on skewed score vectors | Why RRF ($k=60$) beats linear score interpolation |
| **4** | **Two-Tier Redis Semantic Caching** | SHA-256 + Vector Cosine ($\le 0.08$) | Debug `struct.pack('<1536f')` endianness | Sub-20ms cache hits offloading 40–60% token spend |
| **5** | **Distributed Rate Limiting & Lua** | Token Bucket Algorithm in Lua | Flood endpoint to verify HTTP 429 & RFC 7807 | Atomic Lua execution vs MULTI/EXEC transaction limits |
| **6** | **Resilient Routing & Circuit Breakers** | LiteLLM, Tenacity, State Machine | Kill primary API key & watch failover | Nygard 3-state breaker vs naive try/except retries |
| **7** | **Observability, Hardening & Whiteboard** | Prometheus, Nginx, Docker, CI/CD | Execute live 5-minute whiteboard script | Full 24-step socket-to-LLM request trace from memory |

---

## 📅 Session 1: ASGI Architecture, Concurrency & The Fast Pitch

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY1-asgi-foundation.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY1-asgi-foundation.md).
* **Phase 1 & 2:** [01-project-overview.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/01-project-overview.md) and [07-technology-qa-bank.md (Questions 1–7)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/07-technology-qa-bank.md).
* **Focus:** Master the 30-second elevator pitch, ASGI lifecycle (`scope, receive, send`), and structured logging with correlation IDs.

### 2. Tactical Scars & Terminal Drills
* **Start Server:** `uv run uvicorn app.main:app --reload --port 8000`
* **Test Health Endpoint:** `curl -i http://localhost:8000/healthz`
  * Inspect the headers: verify `X-Request-Id` is present.
  * Inspect terminal: observe JSON logs formatted by [app/core/logging.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/logging.py) with correlation context.
* **The Scar Drill (Event Loop Starvation):**
  * Locate [app/middleware/correlation.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/middleware/correlation.py).
  * *Experiment:* What happens if someone runs `time.sleep(2)` inside an async route? Test it mentally or on a scratch route. All concurrent requests freeze. Explain why in interviews: Python's event loop is single-threaded; any synchronous blocking call blocks the entire server.

### 3. The Interview Defense Script
* **Interviewer:** *"Can you describe SentinelRAG in 60 seconds?"*
* **Your Script:**
  > "SentinelRAG is an asynchronous AI middleware gateway built with Python 3.12, FastAPI, and Uvicorn. It sits between client applications and LLM providers like Groq and OpenAI. It solves three critical production problems: cold LLM latency and high token costs via sub-20ms Redis semantic caching; provider outages via an autonomous sliding-window circuit breaker and retry engine; and retrieval mismatches through a single-store PostgreSQL 16 hybrid search engine fusing HNSW vectors and BM25 keywords via Reciprocal Rank Fusion."

### 4. Checkpoint Self-Test
* *Can you explain without hesitation how `asyncio.to_thread` differs from running a coroutine with `await`?* (Answer in Question 2 of [07-technology-qa-bank.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/07-technology-qa-bank.md)).

---

## 📅 Session 2: PostgreSQL 16, pgvector & Single-Store Storage

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY2-database-layer.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY2-database-layer.md).
* **Phase 1 & 2:** [02-architecture-and-contracts.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/02-architecture-and-contracts.md) and [05-tradeoffs-and-system-design.md (Trade-Off #1)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/05-tradeoffs-and-system-design.md).
* **Focus:** Master SQLAlchemy 2.0 Async, `asyncpg` binary protocol, and `pgvector` HNSW index mechanics.

### 2. Tactical Scars & Terminal Drills
* **Inspect Schema:** View [app/models/document.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/models/document.py).
  * Look at line 48: `mapped_column(Vector(1536))`
  * Look at line 52: `mapped_column(TSVECTOR)`
* **Run Database Tests:**
  ```bash
  uv run pytest tests/test_db.py -v
  ```
* **The Scar Drill (HNSW Index Build Memory):**
  * Learn what happens if `maintenance_work_mem` is too small in Postgres: building an HNSW graph on 100k vectors will fail or spill to disk, taking hours instead of minutes.

### 3. The Interview Defense Script
* **Interviewer:** *"Why didn't you use a dedicated vector database like Pinecone, Milvus, or Qdrant?"*
* **Your Script:**
  > "Using a dedicated vector database alongside Postgres creates **multi-database sprawl** and a notorious **dual-write consistency problem**. If a document write succeeds in Postgres but fails in Pinecone, my application state is corrupted unless I introduce an event-driven outbox pattern with Kafka and Debezium. In SentinelRAG, the document text, relational metadata, 1536-dimensional embedding, and full-text `tsvector` all reside in a single PostgreSQL 16 table. We achieve atomic ACID transactions, standard backup/restore via `pg_dump`, and with pgvector's HNSW index ($m=16, ef=64$), we achieve sub-15ms vector query latencies across millions of chunks without maintaining a second distributed cluster."

### 4. Checkpoint Self-Test
* *What is the difference between `<=>`, `<->`, and `<#>` in pgvector?* (`<=>` is cosine distance, `<->` is Euclidean L2 distance, `<#>` is negative inner product).

---

## 📅 Session 3: Hybrid Retrieval & Mathematical Reciprocal Rank Fusion ($k=60$)

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY3-hybrid-search.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY3-hybrid-search.md).
* **Phase 1 & 2:** [03-core-concepts.md (Concepts 3 & 4)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/03-core-concepts.md) and [05-tradeoffs-and-system-design.md (Trade-Off #5)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/05-tradeoffs-and-system-design.md).
* **Focus:** Master why dense search fails on exact keywords, why BM25 fails on semantic concepts, and why RRF solves score normalization.

### 2. Tactical Scars & Terminal Drills
* **Inspect Code:** View [app/repositories/search.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/repositories/search.py) and [app/services/rrf.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/rrf.py).
* **Run Retrieval Tests:**
  ```bash
  uv run pytest tests/test_search.py -v
  ```
* **The Scar Drill (Score Scale Disparity):**
  * Vector cosine similarity yields distances strictly in $[0, 2]$ (or similarity in $[-1, 1]$).
  * PostgreSQL `ts_rank_cd` yields unbounded positive floats (e.g., $0.05$ to $12.4$).
  * If you naively add them: $0.7 \times \text{vector} + 0.3 \times \text{BM25}$, the BM25 score completely dominates or gets silenced depending on document length. RRF solves this by ditching raw scores and using **ordinal ranks**.

### 3. The Interview Defense Script
* **Interviewer:** *"Why Reciprocal Rank Fusion instead of simple weighted score averaging?"*
* **Your Script:**
  > "Dense vector cosine similarity and sparse BM25 scores have fundamentally incompatible distributions. Cosine similarity is bounded, while BM25 is non-linear and unbounded. Any linear combination like $\alpha \cdot S_{dense} + (1-\alpha) \cdot S_{sparse}$ requires fragile min-max normalization that degrades under distribution drift. RRF sidesteps score calibration entirely by operating purely on rank positions: $RRF(d) = \sum \frac{1}{k + r(d)}$. By setting Cormack's empirical constant $k=60$, we ensure top-ranked items receive substantial weight while preventing outliers ranked deep down the candidate lists from skewing the fused results."

### 4. Checkpoint Self-Test
* *What is the score contribution of a document ranked #1 in both dense and sparse lists?*  
  $\frac{1}{60 + 1} + \frac{1}{60 + 1} = \frac{2}{61} \approx 0.0327$.

---

## 📅 Session 4: Two-Tier Redis Semantic Caching & IEEE 754 Float Packing

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY4-semantic-cache-and-rate-limiting.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY4-semantic-cache-and-rate-limiting.md).
* **Phase 1 & 2:** [03-core-concepts.md (Concepts 1 & 2)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/03-core-concepts.md) and [05-tradeoffs-and-system-design.md (Trade-Off #4)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/05-tradeoffs-and-system-design.md).
* **Focus:** Master two-stage caching (exact MD5/SHA-256 followed by vector cosine similarity $\le 0.08$) and binary vector serialization.

### 2. Tactical Scars & Terminal Drills
* **Inspect Code:** View [app/services/cache.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/cache.py).
* **Run Cache Tests:**
  ```bash
  uv run pytest tests/test_cache.py -v
  ```
* **The Scar Drill (IEEE 754 Little-Endian Struct Packing):**
  * Locate where the 1536-float embedding is packed: `struct.pack('<1536f', *embedding)`.
  * *The Bug to Explain:* If you omit `<` (standard little-endian) or serialize as JSON, Redis either receives big-endian bytes (corrupting cosine calculations) or incurs $10\times$ CPU overhead serializing JSON floats.

### 3. The Interview Defense Script
* **Interviewer:** *"Why do you have an exact SHA-256 hash stage if you already have a semantic vector cache?"*
* **Your Script:**
  > "Performing a vector similarity search in Redis via `FT.SEARCH` requires embedding the incoming prompt ($O(N)$ inference cost) and traversing the HNSW vector index in memory ($O(\log M)$ graph hops). For identical queries—which represent over 30% of enterprise traffic—this is wasteful. In SentinelRAG, Stage 1 evaluates an exact SHA-256 hash in Redis in $<1.5\text{ms}$ with zero embedding cost. Only on an exact-hash miss do we generate an embedding and execute Stage 2 vector cosine search with a strict similarity distance threshold ($\le 0.08$). This two-tier design protects our latency budget and saves 40–60% of total LLM token spend."

### 4. Checkpoint Self-Test
* *Why is the cosine distance threshold set to $\le 0.08$ instead of $0.20$?* (At $0.20$, semantically related but factually distinct questions like "How to deploy to AWS?" vs "How to deploy to GCP?" collide and return false cached answers).

---

## 📅 Session 5: Distributed Rate Limiting & Atomic Redis Lua Scripts

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY4-semantic-cache-and-rate-limiting.md (Part B)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY4-semantic-cache-and-rate-limiting.md).
* **Phase 1 & 2:** [03-core-concepts.md (Concept 5)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/03-core-concepts.md) and [05-tradeoffs-and-system-design.md (Trade-Off #2)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/05-tradeoffs-and-system-design.md).
* **Focus:** Master the Token Bucket algorithm, millisecond sliding refills, and Lua script atomicity.

### 2. Tactical Scars & Terminal Drills
* **Inspect Lua Script:** View [app/core/redis.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/redis.py) around line 45 (`RATE_LIMIT_LUA_SCRIPT`).
* **Run Rate Limiting Tests:**
  ```bash
  uv run pytest tests/test_rate_limit.py -v
  ```
* **The Scar Drill (Triggering HTTP 429):**
  * Start the server and send 10 rapid curl requests:
    ```bash
    for i in {1..10}; do curl -i -H "X-API-Key: test_key" http://localhost:8000/api/v1/chat/completions; done
    ```
  * Verify the server returns `HTTP 429 Too Many Requests` with RFC 7807 Problem Details and `Retry-After` headers.

### 3. The Interview Defense Script
* **Interviewer:** *"Why did you use a Redis Lua script instead of standard Redis MULTI/EXEC transactions?"*
* **Your Script:**
  > "A Token Bucket rate limiter requires a **read-modify-write** sequence: we must read the last timestamp and current token count, compute elapsed time, add refilled tokens, check if tokens $\ge 1$, decrement, and write back. In Redis, `MULTI/EXEC` simply queues commands—it cannot make conditional decisions based on intermediate read values within the same transaction. Using Python-side locks would require distributed redlocks across instances, destroying latency. A Redis Lua script executes directly inside Redis's single-threaded C engine, guaranteeing atomic execution in a single network round-trip without concurrency race conditions."

### 4. Checkpoint Self-Test
* *What headers does SentinelRAG return during rate limiting?* (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After`).

---

## 📅 Session 6: Resilient Multi-Provider LLM Routing & Circuit Breaking

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY5-resilient-llm-router.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY5-resilient-llm-router.md).
* **Phase 1 & 2:** [03-core-concepts.md (Concept 6)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/03-core-concepts.md), [05-tradeoffs-and-system-design.md (Trade-Off #3)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/05-tradeoffs-and-system-design.md), and [06-security-and-resilience.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/06-security-and-resilience.md).
* **Focus:** Master the Michael Nygard Circuit Breaker state machine (`CLOSED`, `OPEN`, `HALF_OPEN`), Tenacity exponential retries with jitter, and failover routing.

### 2. Tactical Scars & Terminal Drills
* **Inspect Code:** View [app/core/circuit_breaker.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/core/circuit_breaker.py) and [app/services/router.py](file:///c:/Users/kiran/Desktop/sentinel-rag/app/services/router.py).
* **Run Router Tests:**
  ```bash
  uv run pytest tests/test_router.py -v
  ```
* **The Scar Drill (Simulating Provider Outage):**
  * Set `GROQ_API_KEY=invalid_groq_key` in `.env`.
  * Trigger requests to `/api/v1/chat/completions`.
  * Watch terminal logs: Observe Groq fail with 401/429 $\rightarrow$ Tenacity retries with jitter $\rightarrow$ Circuit Breaker records 5 failures $\rightarrow$ Breaker transitions to `OPEN` $\rightarrow$ Subsequent calls fail-fast in $<2\text{ms}$ and route immediately to OpenAI `gpt-4o-mini`.

### 3. The Interview Defense Script
* **Interviewer:** *"Why not just use LangChain or LlamaIndex for model routing?"*
* **Your Script:**
  > "LangChain and LlamaIndex are bloated application frameworks that introduce deep abstraction layers, heavy dependencies, and non-deterministic overhead unsuitable for high-throughput production gateways. In SentinelRAG, we use **LiteLLM** purely as a clean, standardized translation interface across OpenAI, Groq, and Mistral, wrapped by **Tenacity** for exponential retries with full jitter and a custom **Michael Nygard Circuit Breaker**. When Groq suffers a regional outage or rate limit (HTTP 429), our circuit breaker trips after 5 consecutive failures, failing fast in $<2\text{ms}$ and routing traffic to OpenAI without wasting connection slots waiting for Groq timeouts."

### 4. Checkpoint Self-Test
* *What happens when the Circuit Breaker is in `HALF_OPEN` state?* (It permits a single canary request through; if successful, it resets to `CLOSED`; if it fails, it immediately returns to `OPEN` for another 30 seconds).

---

## 📅 Session 7: Observability, Hardening & 5-Minute Whiteboard Mock Defense

### 1. Objectives & Reading
* **Curriculum:** [PART2-DAY6-telemetry-and-testing.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY6-telemetry-and-testing.md) and [PART2-DAY7-production-hardening-and-deployment.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/archive/curriculum/PART2-DAY7-production-hardening-and-deployment.md).
* **Phase 1 & 2:** [04-codebase-and-trace.md (24-Step Trace)](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-1-implementation/04-codebase-and-trace.md) and [08-whiteboard-and-revision.md](file:///c:/Users/kiran/Desktop/sentinel-rag/notes/phase-2-interview-prep/08-whiteboard-and-revision.md).
* **Focus:** Master the complete 24-step socket-to-LLM trace, Prometheus metric types, Docker multi-stage security, and live whiteboard delivery.

### 2. Tactical Scars & Terminal Drills
* **Inspect Metrics:** Run `curl http://localhost:8000/metrics` and identify:
  * `sentinel_request_duration_seconds_bucket` (Prometheus latency histogram)
  * `sentinel_cache_hits_total` and `sentinel_cache_misses_total`
  * `sentinel_circuit_breaker_state` (0 for CLOSED, 1 for HALF-OPEN, 2 for OPEN)
* **Inspect Production Deployment:** View [Dockerfile](file:///c:/Users/kiran/Desktop/sentinel-rag/Dockerfile) and [nginx/nginx.conf](file:///c:/Users/kiran/Desktop/sentinel-rag/nginx/nginx.conf). Notice non-root user `sentinel:sentinel`, `uv` package caching, and Nginx proxy headers.

### 3. The 5-Minute Whiteboard Final Exam
Take a blank piece of paper or whiteboard. Set a 5-minute timer and sketch this exact topology while speaking:

```
                      ┌───────────────────────────────────────────────┐
                      │              Client Applications              │
                      └───────────────────────┬───────────────────────┘
                                              │ (HTTP / JSON + API Key)
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │         Nginx 1.25 Reverse Proxy              │
                      │  • TLS Termination     • Global Rate Limit   │
                      └───────────────────────┬───────────────────────┘
                                              │ (HTTP / Reverse Proxy)
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │        FastAPI (Uvicorn / uvloop)             │
                      │  • CorrelationIdMiddleware (structlog)        │
                      │  • SecurityMiddleware (API Key validation)    │
                      │  • RateLimitMiddleware (Token-Bucket Lua)     │
                      └───────┬───────────────────────────────┬───────┘
                              │                               │
        ┌─────────────────────┴─────────────┐                 │
        ▼                                   ▼                 ▼
 ┌───────────────┐                   ┌───────────────┐ ┌───────────────┐
 │ Redis 7.2     │                   │ PostgreSQL 16 │ │ LiteLLM       │
 │ • TokenBucket │                   │ • pgvector    │ │ Router        │
 │ • SHA-256 Key │                   │   (HNSW 1536) │ │ • Tenacity    │
 │ • Cosine Match│                   │ • tsvector    │ │ • Circuit     │
 │   (<= 0.08)   │                   │ • RRF (k=60)  │ │   Breaker     │
 └───────────────┘                   └───────────────┘ └───────┬───────┘
                                                               │
                                         ┌─────────────────────┴──────────┐
                                         ▼                                ▼
                                  ┌───────────────┐                ┌───────────────┐
                                  │ Primary: Groq │ ──Failover───> │ Fallback:     │
                                  │ (Llama 3.3)   │                │ OpenAI/Mistral│
                                  └───────────────┘                └───────────────┘
```

*Verbal Trace (The 24-Step Summary):* Recite the socket lifecycle from Uvicorn TCP receive $\rightarrow$ Correlation ID generation $\rightarrow$ Lua token bucket $\rightarrow$ SHA-256 exact cache $\rightarrow$ Redis vector search $\rightarrow$ PostgreSQL HNSW + BM25 search $\rightarrow$ RRF rank fusion $\rightarrow$ LiteLLM Groq call $\rightarrow$ Circuit breaker state check $\rightarrow$ Prometheus telemetry emission.

---

## 🏆 Graduation Self-Scorecard

Before walking into your technical interview, verify you can answer "YES" to all 5:

- [ ] **Can you sketch the whiteboard architecture and speak the 2-minute elevator pitch without pausing?**
- [ ] **Can you explain why PostgreSQL 16 single-store eliminates dual-write corruption compared to Pinecone?**
- [ ] **Can you explain the exact reason Redis Lua scripts guarantee atomic rate limiting where MULTI/EXEC fails?**
- [ ] **Can you describe the little-endian IEEE 754 struct serialization bug and how you resolved it?**
- [ ] **Can you trace a cache-miss request from the incoming TCP socket to the LLM response in under 2 minutes?**
