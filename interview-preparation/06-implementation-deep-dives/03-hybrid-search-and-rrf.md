# Implementation Deep Dive 3: Persistence, Batch Ingestion, Hybrid Search & RRF

**Covered Modules (Verified in [supplementary.md](file:///c:/Users/kiran/Desktop/notes/archive/supplementary.md)):**
`app/db/base.py` | `app/models/document.py` | `app/schemas/document.py` | `app/api/v1/documents.py` | `app/repositories/search.py` | `app/services/rrf.py` | `app/schemas/search.py` | `app/api/v1/search.py` | `alembic/versions/*`

---

## 1. Unified Single-Store Schema (`app/db/base.py`, `app/models/document.py`)

### Mixins (`app/db/base.py`)
- `UUIDPrimaryKeyMixin`: `UUID(as_uuid=True)` with Python-side `default=uuid.uuid4` (avoids guessable auto-increment tenant enumeration; enables knowing UUIDs prior to flush).
- `TimestampMixin`: `DateTime(timezone=True)` using timezone-aware `datetime.now(UTC)` (`datetime.utcnow()` is deprecated in Python 3.12) plus `server_default=func.now()` and `onupdate=func.now()` so raw SQL inserts still receive timestamps.

### `DocumentChunk` (`app/models/document.py`)
Consolidates relational, JSONB, dense vector, and full-text columns in a single `document_chunks` table:
- `tenant_id`: `String(64)`, B-tree indexed (`index=True`).
- `document_name`: `String(255)`.
- `chunk_text`: `Text`.
- `metadata_json`: `JSONB` with `server_default=text("'{}'::jsonb")` + GIN index (`ix_document_chunks_metadata_json`).
- `embedding`: `Vector(1536)` + HNSW index (`ix_document_chunks_embedding_hnsw`, `m=16`, `ef_construction=64`, `vector_cosine_ops`).
- `search_vector`: `TSVECTOR` declared with `Computed("to_tsvector('english', chunk_text)", persisted=True)` + GIN index (`ix_document_chunks_search_vector`).
  - *Crucial SQLAlchemy rule:* Declaring `Computed(..., persisted=True)` tells SQLAlchemy that PostgreSQL owns this column, so ORM/bulk `INSERT` statements automatically omit `search_vector`.

### Alembic Migrations (`alembic/versions/`)
- `0001_create_document_chunks.py`: Executes `CREATE EXTENSION IF NOT EXISTS vector;` before creating `document_chunks` and all four indexes.
- `0002_add_hybrid_search_indexes.py`: Demonstrates idempotent `CREATE INDEX IF NOT EXISTS` functional GIN + HNSW index DDL in `upgrade()`/`downgrade()`.

---

## 2. Batch Document Ingestion (`app/api/v1/documents.py`)

`POST /api/v1/documents/batch` (returns `201 Created`):
1. `BatchIngestRequest` validates `tenant_id` (`1..64` chars) and `chunks` (`1..1000` items), where each `ChunkCreate.embedding` enforces `min_length=1536, max_length=1536` at the Pydantic Rust layer.
2. Maps chunks into a list of dicts (excluding `search_vector`).
3. Executes a single atomic multi-row PostgreSQL bulk insert:
   ```python
   stmt = insert(DocumentChunk).returning(DocumentChunk.id)
   result = await session.scalars(stmt, chunk_records)
   chunk_ids = list(result.all())
   await session.commit()
   ```
   - Uses `sqlalchemy.dialects.postgresql.insert` (not generic `sqlalchemy.insert`) specifically to access PostgreSQL's `RETURNING` clause in one round-trip.
   - Explicit `await session.commit()` is required because `get_async_session` only rolls back on error and closes in `finally`.

---

## 3. Hybrid Retrieval Repository (`app/repositories/search.py`)

### Memory-Optimized Candidate Containers
`DenseCandidate` (`chunk`, `distance`, `similarity_score`, `rank`) and `SparseCandidate` (`chunk`, `lexical_score`, `rank`) both use `@dataclass(slots=True)`—replacing per-instance `__dict__` hash tables with fixed C pointer slots, saving ~60% per-object RAM and reducing GC pressure when constructing 100+ candidates per request.

### Dense Search (`search_dense`)
- Builds `distance_expr = DocumentChunk.embedding.cosine_distance(query_vector).label("distance")` (SQL `<=>`), filtered by `tenant_id`, ordered `.asc()`, limited to `limit` (default `50`).
- Converts distance $d \in [0, 2]$ to similarity bounded at zero: `similarity = max(0.0, 1.0 - float(dist))`.

### Sparse Search (`search_sparse`)
- Guard: returns `[]` immediately if `query_text.strip()` is empty.
- Uses `func.websearch_to_tsquery("english", sanitized_query)` (safe against syntax errors on punctuation/operators unlike `to_tsquery`) matched via `.op("@@")(ts_query)` against the GIN-indexed `search_vector`.
- Ranks by descending `func.ts_rank_cd(...)` (Cover Density, scoring proximity of matching terms).

### Sequential Execution Rationale (`search_hybrid_candidates`)
```python
dense_candidates = await self.search_dense(...)
sparse_candidates = await self.search_sparse(...)
return dense_candidates, sparse_candidates
```
- **Why not `asyncio.gather`?** A SQLAlchemy `AsyncSession` binds to a single `asyncpg` TCP socket. Calling `execute()` concurrently on one session triggers `asyncpg.exceptions.InterfaceError: cannot perform operation: another operation is in progress`. Running sequentially takes only ~10–25ms total without checking out double the DB pool connections per request.

---

## 4. Reciprocal Rank Fusion (`app/services/rrf.py` & `app/api/v1/search.py`)

### Two Functions in `rrf.py`
1. `reciprocal_rank_fusion(rankings: Sequence[Sequence[T]], k: int = 60)`: Generic `TypeVar("T", bound=Hashable)` pure algorithm for arbitrary ID lists. Validates `k >= 1`, sums `1.0 / (k + rank_idx)` using **1-based** enumeration (`enumerate(ranking, start=1)`).
2. `fuse_hybrid_candidates(dense_candidates, sparse_candidates, k=60, top_k=None) -> list[FusedSearchResult]`:
   - Accumulates scores keyed by `cand.chunk.id` (UUID deduplication across both lists).
   - Preserves original `dense_rank`, `dense_score`, `sparse_rank`, and `sparse_score` for response telemetry.
   - **Deterministic Tie-Breaking:**
     ```python
     fused_results.sort(
         key=lambda r: (
             r.fused_score,
             r.dense_score if r.dense_score is not None else -1.0,
             r.sparse_score if r.sparse_score is not None else -1.0,
         ),
         reverse=True,
     )
     ```
     If Chunk 1 is (#1 dense, #2 sparse) and Chunk 2 is (#2 dense, #1 sparse), their RRF scores tie ($\frac{1}{61}+\frac{1}{62}$)—Python deterministically picks the higher `dense_score`, then `sparse_score`.

### Standalone Endpoint (`POST /api/v1/search/hybrid`)
Exposes `SearchRepository` + `fuse_hybrid_candidates` directly with configurable `top_k` (`1..100`, default `10`), `dense_limit` (`50`), `sparse_limit` (`50`), and `rrf_k` (`60`)—returning `HybridSearchResponse` with rounded component scores and `duration_ms`.

---
**Next:** [04-resilient-router-chat-and-analytics.md](file:///c:/Users/kiran/Desktop/notes/interview-preparation/06-implementation-deep-dives/04-resilient-router-chat-and-analytics.md)

