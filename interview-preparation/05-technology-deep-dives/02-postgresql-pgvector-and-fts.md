# Technology Deep Dive 2: PostgreSQL 16, `pgvector` (HNSW), Full-Text Search & Async SQLAlchemy

## 1. Fundamental Concepts & Internals

### `asyncpg` & SQLAlchemy 2.0 Async
- **Why `asyncpg` is fast:** Unlike `psycopg2`/`psycopg3` (built around C `libpq`), `asyncpg` implements PostgreSQL's wire protocol from scratch in Cython, speaking native binary format (zero text-to-float/UUID parsing overhead) and caching prepared statements per connection.
- **Process-per-Connection Reality:** Every PostgreSQL connection spawns an OS `postgres` backend process (~5–10 MB RAM + buffer locks). Opening 1,500 direct app connections across pods causes catastrophic context switching—hence keeping pools modest per worker (`pool_size=20, max_overflow=10`) and inserting **PgBouncer** (transaction pooling mode) at multi-pod scale.
  - *PgBouncer Transaction Pooling Gotcha:* You must disable `asyncpg` prepared statement caching (`statement_cache_size=0`) when running behind PgBouncer in transaction mode because server connections swap between clients after each `COMMIT`.

### `pgvector` Distance Operators & Index Types
| Operator | Metric | Formula | When Identical? |
| :--- | :--- | :--- | :--- |
| `<=>` | **Cosine Distance** (used in SentinelRAG) | $1 - \frac{u \cdot v}{\|u\|\|v\|}$ | Scale-invariant; ideal for text embeddings. |
| `<->` | **Euclidean ($L_2$) Distance** | $\sqrt{\sum (u_i - v_i)^2}$ | Monotonically equal to cosine for unit vectors ($\|u\|=1$). |
| `<#>` | **Negative Inner Product** | $-\sum u_i v_i$ | 2–3x faster if vectors are pre-normalized (avoids norm/sqrt at query time). |

- **IVFFlat vs. HNSW:**
  - *IVFFlat:* K-means clusters vectors into Voronoi `lists`; searches top `probes` cells. Low RAM, fast build, **bad for SentinelRAG** because it requires training on representative existing data (useless on empty tables) and recall degrades as new chunks shift distributions.
  - *HNSW (Hierarchical Navigable Small World — chosen):* Multi-layer proximity skip-list graph. Upper layers carry sparse long-range links; Layer 0 holds dense local neighbors. Greedy traversal runs in $O(\log N)$, supports live incremental inserts with zero retraining, and achieves $>95\text{--}98\%$ recall at sub-15ms query latency—at the cost of higher RAM and slower index builds (`maintenance_work_mem` must be tuned high enough to keep graph construction in RAM).
- **HNSW Parameters:**
  - `m = 16`: Max bidirectional edges per node per layer (memory vs. recall).
  - `ef_construction = 64`: Candidate priority-queue depth during index build (build time vs. graph quality).
  - `hnsw.ef_search` (runtime session GUC, default 40): Search queue depth per query (latency vs. recall).

### PostgreSQL Native Full-Text Search (`tsvector` + `GIN` + `ts_rank_cd`)
- **`to_tsvector('english', chunk_text)`:** Tokenizes text, drops English stop words (`the`, `is`), stems lexemes (`running` $\to$ `run`), and stores sorted `(lexeme, positions)` pairs.
- **GIN (Generalized Inverted Index):** B-tree over unique lexemes whose leaf nodes point to posting lists/trees of physical heap Tuple IDs (`TID`s).
- **`websearch_to_tsquery` vs `to_tsquery`:** `to_tsquery` throws syntax errors on raw user input (`C++`, unclosed quotes, `&`); `websearch_to_tsquery` safely parses Google-like input (phrases in quotes, `-` negation, spaces as `AND`).
- **`ts_rank_cd` vs `ts_rank`:** `ts_rank_cd` computes **Cover Density**—rewarding documents where query terms appear close together in the same sentence/window rather than scattered apart.

---

## 2. Why Selected for SentinelRAG (Single-Store Architecture)
SentinelRAG stores text, JSONB metadata, `Vector(1536)` embeddings, and `TSVECTOR` in **one `document_chunks` table** inside PostgreSQL 16 rather than splitting Postgres + Pinecone/Qdrant + Elasticsearch.
- **Eliminates Dual-Write Drift:** Deleting or updating a document updates relational metadata, dense index, and lexical index in one ACID transaction—no Kafka/Debezium outbox needed.
- **Unified Tenant Filtering:** `WHERE tenant_id = :tenant_id` runs natively inside Postgres alongside B-tree/GIN/HNSW indexes.
- **Operational Simplicity:** One backup (`pg_dump`), one connection pool, zero external vector SaaS egress hops.

---

## 3. Project Implementation & Configuration Details

### A. Engine & Session Tuning (`app/db/session.py`)
```python
engine: AsyncEngine = create_async_engine(
    url=settings.DATABASE_URL,
    pool_size=20, max_overflow=10,
    pool_recycle=300,       # Recycle every 5m to drop firewall-killed sockets
    pool_pre_ping=True,     # Lightweight SELECT 1 on checkout
    pool_timeout=30,
)
async_session_factory = async_sessionmaker(
    bind=engine, class_=AsyncSession,
    expire_on_commit=False, # Prevents MissingGreenlet after commit()
    autoflush=False,
)
```
- **Why `expire_on_commit=False`?** Default SQLAlchemy expires attributes on `commit()`, triggering synchronous lazy-load SQL on next attribute access (`chunk.id`). In async SQLAlchemy, implicit lazy loads fail with `MissingGreenlet` / `DetachedInstanceError`.

### B. `DocumentChunk` Schema & Computed Column (`app/models/document.py`)
- Inherited mixins (`app/db/base.py`): `UUIDPrimaryKeyMixin` (UUIDv4 prevents multi-tenant sequential ID enumeration like `/docs/1,2,3`, trading 16-byte random B-tree locality for security) and `TimestampMixin` (`datetime.now(UTC)` + `server_default=func.now()` covering both ORM and raw SQL writes).
- `search_vector`: Declared as `Computed("to_tsvector('english', chunk_text)", persisted=True)`. PostgreSQL maintains it in C automatically; SQLAlchemy omits it from `INSERT`/`UPDATE` (trying to insert a value fails with `cannot insert a non-DEFAULT value into column "search_vector"`).
- **4 Indexes on `document_chunks`:**
  1. `ix_document_chunks_tenant_id` (B-Tree on `tenant_id`)
  2. `ix_document_chunks_metadata_json` (GIN on `metadata_json` JSONB for `@>` containment filters)
  3. `ix_document_chunks_search_vector` (GIN on `search_vector`)
  4. `ix_document_chunks_embedding_hnsw` (HNSW on `embedding` with `m=16, ef_construction=64, vector_cosine_ops`)

### C. Batch Ingestion (`app/api/v1/documents.py`)
- Uses `sqlalchemy.dialects.postgresql.insert(DocumentChunk).returning(DocumentChunk.id)` (generic `sqlalchemy.insert` lacks PostgreSQL `RETURNING`) to insert up to `1000` chunks atomically and return generated UUIDs in one DB round-trip.

### D. Sequential Hybrid Retrieval (`app/repositories/search.py`)
- `SearchRepository.search_hybrid_candidates` executes `search_dense` (`embedding.cosine_distance(query_vector)`, returning `DenseCandidate(..., similarity_score=max(0.0, 1.0 - dist))`) **then** `search_sparse` **sequentially** on the single `AsyncSession`.
- **Why NOT `asyncio.gather(search_dense(), search_sparse())`?** An `AsyncSession` holds a single half-duplex `asyncpg` TCP socket. Concurrent `execute()` calls interleave wire frames and crash immediately with `asyncpg.exceptions.InterfaceError: cannot perform operation: another operation is in progress`.
- **Why Two Separate Queries Instead of One Combined `WHERE tsv @@ ... ORDER BY embedding <=>`?** Postgres cannot traverse both a GIN index and an HNSW index in a single physical scan. If it picks HNSW and filters rare keywords post-scan, latency explodes scanning non-matching nodes; if it picks GIN, it brute-force sorts vectors without HNSW. Running two pure index scans + RRF keeps both under ~10–15ms.

---

## 4. Practical Problems, Trade-Offs & Scaling Limits
1. **Operator Mismatch Silent `Seq Scan`:** Querying `<->` (L2) on an index built with `vector_cosine_ops` forces PostgreSQL to skip HNSW and scan every row on disk.
2. **Database Migrations (`alembic upgrade head`):** Always run migrations as a single pre-deployment CI/CD step (`docker run --rm ... alembic upgrade head`), **never** inside container entrypoint scripts where 4 starting pods race on DDL locks and need elevated DDL privileges.
3. **Scaling Ceiling:** Single-node `pgvector` HNSW shines up to **~10M–20M vectors** (as long as the HNSW graph + vectors fit in RAM). Beyond that—or at $>10\text{k}$ chunk writes/sec causing WAL write-amplification and autovacuum overhead—add read replicas for `SearchRepository`, use `halfvec` (16-bit floats, 50% RAM cut), or decouple vectors to Qdrant/Milvus.

---

## 5. Quick Interview Questions
1. *Why does `asyncio.gather` fail on one `AsyncSession`?*
2. *Explain why `expire_on_commit=False` and `pool_pre_ping=True` are configured in `app/db/session.py`.*
3. *Why can't Postgres efficiently execute `WHERE search_vector @@ ... ORDER BY embedding <=> ... LIMIT 10` in one query?*
4. *At what scale would you leave single-store `pgvector` for Qdrant/Milvus, and what intermediate steps (read replicas, `halfvec`, PgBouncer) come first?*

