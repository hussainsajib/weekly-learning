# Week 6 — RAG & Vector Databases

**Week of:** July 13, 2026
**Estimated study time:** ~2 hours
**Tags:** `ai` `rag` `databases` `vectors`

---

## Overview

Retrieval-Augmented Generation (RAG) is an architecture pattern that addresses one of the core limitations of large language models: their knowledge is frozen at training time and they cannot reason over private, domain-specific, or recently updated data. RAG solves this by splitting the problem into two phases — first *retrieve* the most relevant context from a corpus of documents, then *generate* a response conditioned on that retrieved context. The retrieval step is powered by vector similarity search, which finds semantically related chunks even when keyword overlap is absent. For an engineer like you working on the CRM-EHR Integration Platform middleware, this means you could build a system that answers questions about EHR system policy types, Salesforce object schemas, or ETL job history without fine-tuning a model or stuffing a 100,000-token prompt.

Vector databases store high-dimensional numerical representations of text (embeddings) and answer approximate-nearest-neighbor queries against them in milliseconds. The key insight is that an embedding model maps semantically similar phrases to nearby points in vector space — so "property damage liability coverage" and "PDL policy" end up close together even though they share no words. This property enables a retrieval layer that can bridge the gap between how humans phrase questions and how documents are actually written, which is especially valuable in domains like insurance EHR integration where jargon is dense and inconsistent.

The architecture has several moving parts that interact: a chunking pipeline that splits source documents into embedding-sized pieces, an embedding model that converts those chunks to vectors, a vector store that indexes and serves similarity queries, and an LLM that synthesizes a final answer from the retrieved chunks plus the original question. Each of these components has its own quality/performance tradeoffs, and getting them right requires understanding the full pipeline end-to-end. A weak link anywhere — over-aggressive chunking, a mismatched embedding model, poorly tuned HNSW parameters, or an LLM with too small a context window — will degrade the final answer quality.

For the integration platform specifically, you already have PostgreSQL infrastructure with the `pgvector` extension available. This means you can add semantic search capabilities to the existing middleware without introducing a new managed service. A well-designed RAG layer over your EHR system schema documentation, Salesforce knowledge base articles, and policy type definitions could dramatically reduce the time engineers spend context-switching between documentation systems when debugging sync failures or writing new trigger handlers.

---

## 1. RAG Architecture End-to-End

The RAG pipeline splits cleanly into an **indexing path** (offline, run when documents change) and a **query path** (online, run per user request).

### Indexing path

```
Raw documents → Loader → Chunker → Embedding model → Vector store (upsert)
```

### Query path

```
User question → Embedding model → Vector store (similarity search) → Top-k chunks
    → Prompt assembly → LLM → Answer
```

The indexing path runs as a batch job or incremental pipeline. In the integration platform context you could trigger re-indexing whenever a new Salesforce object schema is deployed or a new EHR system endpoint is documented. The query path runs synchronously inside the FastAPI middleware — a new `/search` or `/ask` endpoint that wraps both retrieval and generation.

### Minimal FastAPI RAG endpoint

```python
# app/api/v2/rag.py
from __future__ import annotations

import asyncio
from typing import Annotated

import httpx
from fastapi import APIRouter, Depends, Query
from pgvector.sqlalchemy import Vector
from sqlalchemy import select, text
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.db import get_session
from app.models.documents import DocumentChunk

router = APIRouter(prefix="/rag", tags=["rag"])


async def embed(text: str) -> list[float]:
    """Call a local or remote embedding endpoint."""
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            "http://embedding-service/embed",
            json={"input": text},
            timeout=10.0,
        )
        resp.raise_for_status()
        return resp.json()["embedding"]


@router.get("/search")
async def semantic_search(
    q: Annotated[str, Query(min_length=3, max_length=512)],
    top_k: int = 5,
    session: AsyncSession = Depends(get_session),
) -> list[dict]:
    query_vec = await embed(q)
    stmt = (
        select(
            DocumentChunk.id,
            DocumentChunk.content,
            DocumentChunk.source,
            DocumentChunk.embedding.cosine_distance(query_vec).label("distance"),
        )
        .order_by(text("distance"))
        .limit(top_k)
    )
    rows = (await session.execute(stmt)).all()
    return [
        {"id": r.id, "content": r.content, "source": r.source, "score": 1 - r.distance}
        for r in rows
    ]
```

**Common mistake:** Running the embedding call inside a synchronous SQLAlchemy session callback. Always `await` embedding I/O before opening the database transaction so you don't hold a connection while waiting on an external HTTP call.

---

## 2. Chunking Strategies

Chunking is the step that most directly controls retrieval quality. A chunk is the unit of retrieval — if a chunk is too large it contains noise that dilutes the embedding; if it's too small it loses the surrounding context the LLM needs to answer the question.

### Fixed-size chunking

The simplest approach: split every N tokens with an overlap of M tokens. Overlap ensures that facts spanning a chunk boundary are not lost.

```python
from typing import Iterator


def fixed_size_chunks(
    text: str,
    chunk_size: int = 512,
    overlap: int = 64,
    tokenizer=None,
) -> Iterator[str]:
    """Yield overlapping fixed-size token chunks."""
    tokens = tokenizer.encode(text) if tokenizer else text.split()
    start = 0
    while start < len(tokens):
        end = min(start + chunk_size, len(tokens))
        chunk = tokens[start:end]
        yield tokenizer.decode(chunk) if tokenizer else " ".join(chunk)
        if end == len(tokens):
            break
        start += chunk_size - overlap
```

### Semantic / recursive chunking

Split on natural boundaries first (paragraphs → sentences → words), only falling back to hard splits when a section exceeds the size budget. This keeps paragraphs together which improves embedding coherence.

```python
import re


def recursive_chunks(
    text: str,
    max_chars: int = 1500,
    separators: list[str] | None = None,
) -> list[str]:
    separators = separators or ["\n\n", "\n", ". ", " "]
    for sep in separators:
        parts = text.split(sep)
        chunks: list[str] = []
        current = ""
        for part in parts:
            candidate = (current + sep + part).strip() if current else part.strip()
            if len(candidate) <= max_chars:
                current = candidate
            else:
                if current:
                    chunks.append(current)
                current = part.strip()
        if current:
            chunks.append(current)
        if all(len(c) <= max_chars for c in chunks):
            return chunks
    return [text[i : i + max_chars] for i in range(0, len(text), max_chars)]
```

### Document-aware chunking (for the integration platform)

EHR system schema documentation and Salesforce field definitions have structured formats. Prefer chunking at object/field boundaries rather than arbitrary token counts:

```python
import json
from dataclasses import dataclass


@dataclass
class SchemaChunk:
    object_name: str
    field_name: str | None
    content: str
    metadata: dict


def chunk_salesforce_schema(schema_json: dict) -> list[SchemaChunk]:
    chunks: list[SchemaChunk] = []
    for obj_name, obj_def in schema_json["objects"].items():
        # One chunk per object header
        header = f"Object: {obj_name}\nLabel: {obj_def['label']}\nDescription: {obj_def.get('description', '')}"
        chunks.append(SchemaChunk(obj_name, None, header, {"type": "object_header"}))
        # One chunk per field
        for field in obj_def.get("fields", []):
            field_text = (
                f"Object: {obj_name}\n"
                f"Field: {field['name']} ({field['type']})\n"
                f"Label: {field.get('label', '')}\n"
                f"Description: {field.get('description', '')}"
            )
            chunks.append(SchemaChunk(obj_name, field["name"], field_text, {"type": "field"}))
    return chunks
```

**Common mistake:** Forgetting to include metadata (source file, object name, section heading) alongside the chunk text. Without metadata, retrieved chunks cannot be cited or filtered, and the LLM cannot tell the user *where* the answer came from.

---

## 3. Embedding Models — Dense vs. Sparse

An embedding model maps text to a fixed-length float vector. The choice of model sets a ceiling on retrieval quality.

| Property | Dense embeddings | Sparse embeddings (BM25/SPLADE) |
|---|---|---|
| Representation | 768–1536 floats | Vocabulary-sized sparse vector |
| Handles synonyms | Yes | Poorly |
| Handles rare keywords | Poorly | Excellent |
| Index size | O(n × d) | O(nnz) — usually smaller |
| Latency | ANN query (fast) | Inverted index (fast) |
| Examples | `text-embedding-3-small`, `all-MiniLM-L6-v2` | BM25, SPLADE, `opensearch` |

**Dense models** are the default for RAG because they capture semantic meaning. For the integration platform you want a model trained on a domain close to insurance/healthcare. `text-embedding-3-small` (OpenAI) or `BAAI/bge-base-en-v1.5` (open-source, 768-dim) are strong baselines.

**Embedding dimensions** affect both quality and storage. 1536-dim embeddings are more expressive but require more RAM and disk. For pgvector on PostgreSQL, a 1536-dim vector index on 500,000 rows consumes roughly 3 GB of memory. Start with 768-dim if resources are constrained.

```python
# app/services/embedder.py
from __future__ import annotations

import numpy as np
from sentence_transformers import SentenceTransformer


class Embedder:
    _model: SentenceTransformer | None = None

    def __init__(self, model_name: str = "BAAI/bge-base-en-v1.5"):
        self.model_name = model_name

    @property
    def model(self) -> SentenceTransformer:
        if self._model is None:
            self._model = SentenceTransformer(self.model_name)
        return self._model

    def embed(self, texts: list[str]) -> list[list[float]]:
        vecs: np.ndarray = self.model.encode(
            texts,
            normalize_embeddings=True,   # required for cosine similarity via dot product
            batch_size=32,
            show_progress_bar=False,
        )
        return vecs.tolist()
```

**Common mistake:** Not normalizing embeddings before storing them. If you use `<=>` (cosine distance) in pgvector, the math is equivalent to dot product on normalized vectors — which means un-normalized vectors will give wrong similarity scores. Always normalize at embed time, not query time, so stored vectors are already unit-length.

---

## 4. pgvector — RAG on Your Existing PostgreSQL

Since the integration platform already runs PostgreSQL (Cloud SQL or Kubernetes-hosted), `pgvector` is the lowest-friction path to production vector search. You get ACID transactions, row-level security, and the ability to JOIN vector results with your existing relational tables in a single query.

### Schema with Alembic migration

```python
# alembic/versions/0010_add_document_chunks.py
from alembic import op
import sqlalchemy as sa
from pgvector.sqlalchemy import Vector

def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")
    op.create_table(
        "document_chunks",
        sa.Column("id", sa.UUID, primary_key=True, server_default=sa.text("gen_random_uuid()")),
        sa.Column("source", sa.Text, nullable=False),
        sa.Column("content", sa.Text, nullable=False),
        sa.Column("embedding", Vector(768), nullable=False),
        sa.Column("metadata", sa.JSONB, nullable=False, server_default="{}"),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.text("now()")),
    )
    # HNSW index — better query-time performance than IVFFlat for most RAG workloads
    op.execute(
        "CREATE INDEX document_chunks_embedding_hnsw "
        "ON document_chunks USING hnsw (embedding vector_cosine_ops) "
        "WITH (m = 16, ef_construction = 64)"
    )

def downgrade() -> None:
    op.drop_table("document_chunks")
```

### SQLAlchemy model

```python
# app/models/documents.py
from __future__ import annotations

import uuid
from datetime import datetime

from pgvector.sqlalchemy import Vector
from sqlalchemy import DateTime, Text, func
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class DocumentChunk(Base):
    __tablename__ = "document_chunks"

    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    source: Mapped[str] = mapped_column(Text, nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    embedding: Mapped[list[float]] = mapped_column(Vector(768), nullable=False)
    metadata: Mapped[dict] = mapped_column(JSONB, nullable=False, default=dict)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

### Upsert pattern (idempotent indexing)

```python
# app/services/indexer.py
from __future__ import annotations

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.documents import DocumentChunk
from app.services.embedder import Embedder

embedder = Embedder()


async def upsert_chunks(
    chunks: list[dict],  # [{"source": ..., "content": ..., "metadata": ...}]
    session: AsyncSession,
) -> int:
    texts = [c["content"] for c in chunks]
    vectors = embedder.embed(texts)
    records = [
        DocumentChunk(
            source=c["source"],
            content=c["content"],
            embedding=v,
            metadata=c.get("metadata", {}),
        )
        for c, v in zip(chunks, vectors)
    ]
    session.add_all(records)
    await session.commit()
    return len(records)
```

**Common mistake:** Creating an IVFFlat index before the table has enough rows. IVFFlat requires a `VACUUM ANALYZE` and at least `lists * 10` rows to build meaningful centroids. For small datasets (< 10,000 rows) skip the index entirely or use HNSW, which works well at any scale.

---

## 5. HNSW and IVFFlat — How ANN Indexes Work

Both indexes trade a small amount of recall (might miss the true nearest neighbor) for a large speedup over exact brute-force search.

### IVFFlat (Inverted File with Flat quantization)

1. At build time, cluster all vectors into `lists` centroids using k-means.
2. At query time, search only the `probes` nearest centroids instead of all `lists`.
3. Exact distance to every vector in those `probes` buckets.

```sql
-- Build
CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Tune probes at query time (higher = more recall, slower)
SET ivfflat.probes = 10;
```

Rule of thumb: `lists ≈ sqrt(n)` for balanced build/query tradeoff. `probes = lists / 10` gives ~95% recall for most distributions.

### HNSW (Hierarchical Navigable Small World)

HNSW builds a layered graph where each node links to its `m` nearest neighbors. Queries traverse from the top (sparse) layer down to the bottom (dense) layer, pruning the search space at each hop.

```sql
-- Build: m controls graph connectivity, ef_construction controls build recall
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Tune ef_search at query time
SET hnsw.ef_search = 100;
```

| Parameter | Effect | Typical range |
|---|---|---|
| `m` | Edges per node — higher = better recall, more memory | 8–64 |
| `ef_construction` | Build-time beam width — higher = better index quality, slower build | 32–128 |
| `ef_search` | Query-time beam width — higher = better recall, slower query | 40–200 |

**HNSW vs IVFFlat for the integration platform:** HNSW is preferred for the RAG use case because it maintains good recall as you add new documents (no need to rebuild the index), whereas IVFFlat degrades and eventually needs a full REINDEX as data grows. The memory overhead (~1.3× the vector data size for default parameters) is acceptable for a few hundred thousand policy documents.

**Common mistake:** Setting `ef_construction` too low to save build time, then being surprised by poor recall. The index build is a one-time cost — spend it. A well-built HNSW index with `m=16, ef_construction=128` has 10–30× better recall at a given query latency than one built with defaults.

---

## 6. Hybrid Search — Combining Dense and Sparse Retrieval

Neither dense nor sparse retrieval is universally better. Dense retrieval wins on semantic similarity; sparse (keyword) retrieval wins on exact term matching, which matters when users search for specific EHR system field names, Salesforce API names, or error codes.

Hybrid search runs both retrievers and fuses the result lists using **Reciprocal Rank Fusion (RRF)** or a weighted score combination.

### RRF implementation

```python
from collections import defaultdict


def reciprocal_rank_fusion(
    result_lists: list[list[str]],  # each inner list: doc IDs in rank order
    k: int = 60,
) -> list[tuple[str, float]]:
    """
    RRF score: sum(1 / (k + rank)) across all result lists.
    k=60 is the standard default from the original RRF paper.
    """
    scores: dict[str, float] = defaultdict(float)
    for results in result_lists:
        for rank, doc_id in enumerate(results, start=1):
            scores[doc_id] += 1.0 / (k + rank)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

### Hybrid search endpoint (pgvector + full-text search)

```python
@router.get("/hybrid-search")
async def hybrid_search(
    q: Annotated[str, Query(min_length=3, max_length=512)],
    top_k: int = 5,
    alpha: float = 0.5,   # weight for dense score vs keyword score
    session: AsyncSession = Depends(get_session),
) -> list[dict]:
    query_vec = await embed(q)

    # Dense retrieval
    dense_stmt = (
        select(DocumentChunk.id, DocumentChunk.content, DocumentChunk.source)
        .order_by(DocumentChunk.embedding.cosine_distance(query_vec))
        .limit(top_k * 3)
    )
    dense_rows = (await session.execute(dense_stmt)).all()

    # Sparse retrieval via PostgreSQL full-text search
    sparse_stmt = (
        select(DocumentChunk.id, DocumentChunk.content, DocumentChunk.source)
        .where(
            DocumentChunk.content.op("@@")(func.plainto_tsquery("english", q))
        )
        .order_by(
            func.ts_rank(
                func.to_tsvector("english", DocumentChunk.content),
                func.plainto_tsquery("english", q),
            ).desc()
        )
        .limit(top_k * 3)
    )
    sparse_rows = (await session.execute(sparse_stmt)).all()

    dense_ids = [str(r.id) for r in dense_rows]
    sparse_ids = [str(r.id) for r in sparse_rows]

    fused = reciprocal_rank_fusion([dense_ids, sparse_ids])
    top_ids = [doc_id for doc_id, _ in fused[:top_k]]

    all_rows = {str(r.id): r for r in [*dense_rows, *sparse_rows]}
    return [
        {"id": doc_id, "content": all_rows[doc_id].content, "source": all_rows[doc_id].source}
        for doc_id in top_ids
        if doc_id in all_rows
    ]
```

**Common mistake:** Normalizing RRF scores to `[0, 1]` and then interpreting them as probabilities. RRF scores are rank aggregation signals, not calibrated confidence values. Use them only for ordering, not for thresholding or display.

---

## 7. Managed Vector Databases — Pinecone, Weaviate, Chroma

When pgvector isn't the right fit (e.g., you need multi-tenant vector isolation, geo-distributed indexes, or billion-scale vectors), managed vector databases offer purpose-built solutions.

| Feature | pgvector | Pinecone | Weaviate | Chroma |
|---|---|---|---|---|
| Deployment | Self-hosted / Cloud SQL | Managed cloud | Self-hosted or cloud | Local or managed |
| Scale ceiling | ~10M vectors (practical) | Billions | Hundreds of millions | Millions |
| Hybrid search | Manual (FTS + pgvector) | Built-in (sparse-dense) | Built-in (BM25 + vector) | Dense only |
| ACID transactions | Yes | No | No | No |
| Cost at 1M vectors | Infrastructure only | ~$70/mo (starter) | Infrastructure or ~$25/mo | Free (local) |
| Integration platform fit | Best — reuse existing infra | Good for production scale | Good — built-in RAG modules | Good for local dev/testing |

### Chroma — local development and testing

Chroma is the fastest way to prototype RAG locally without spinning up PostgreSQL.

```python
# scripts/prototype_rag.py — local dev only, not for production
import chromadb
from chromadb.utils.embedding_functions import SentenceTransformerEmbeddingFunction

client = chromadb.Client()
ef = SentenceTransformerEmbeddingFunction(model_name="BAAI/bge-base-en-v1.5")
collection = client.create_collection("platform_docs", embedding_function=ef)

# Index some integration platform policy type descriptions
collection.add(
    documents=[
        "APP__Policy_Type__c represents the category of insurance policy in the EHR system.",
        "The Line of Business field maps to the EHR system's LOB code in the BDE backend.",
        "Opportunity sync failure usually indicates a missing EHR client ID on the Account.",
    ],
    ids=["pt-1", "lob-1", "opp-err-1"],
)

results = collection.query(query_texts=["why does opportunity sync fail?"], n_results=2)
print(results["documents"])
```

**Common mistake:** Using Chroma (in-memory mode) in a FastAPI app under load. Chroma's default in-memory client is not thread-safe and has no persistence. For production, always use pgvector with your existing PostgreSQL cluster, or a properly deployed Weaviate/Pinecone instance.

---

## 8. Evaluating RAG Quality

RAG quality is measured across two dimensions: **retrieval quality** (did we fetch the right chunks?) and **generation quality** (did the LLM produce a correct, grounded answer?).

### Retrieval metrics

| Metric | What it measures | Formula |
|---|---|---|
| Recall@k | Fraction of relevant docs retrieved in top-k | `|relevant ∩ retrieved| / |relevant|` |
| Precision@k | Fraction of retrieved docs that are relevant | `|relevant ∩ retrieved| / k` |
| MRR | Position of first relevant doc | `mean(1 / rank_of_first_relevant)` |
| NDCG@k | Quality-weighted ranking | Graded relevance, position-discounted |

### Generation metrics

| Metric | What it measures | Tool |
|---|---|---|
| Faithfulness | Answer only uses facts from retrieved chunks | RAGAS, TruLens |
| Answer relevance | Answer addresses the actual question | RAGAS |
| Context precision | Retrieved chunks are relevant to the question | RAGAS |
| Context recall | Retrieved chunks cover all needed facts | RAGAS |

### RAGAS evaluation pipeline

```python
# scripts/evaluate_rag.py
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import answer_relevancy, context_precision, context_recall, faithfulness

samples = [
    {
        "question": "What Salesforce object stores EHR system policy types?",
        "answer": "APP__Policy_Type__c stores EHR system policy types in Salesforce.",
        "contexts": [
            "APP__Policy_Type__c represents the category of insurance policy synced from the EHR system.",
            "Policy types are synced one-way from the EHR system to Salesforce via the ETL pipeline.",
        ],
        "ground_truth": "The APP__Policy_Type__c object stores EHR system policy types.",
    }
]

dataset = Dataset.from_list(samples)
results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)
print(results)
```

**Common mistake:** Evaluating only generation quality and ignoring retrieval quality. A high-faithfulness score with low recall@k just means the LLM accurately described irrelevant chunks. Always measure both retrieval and generation metrics separately — a drop in faithfulness often means retrieval degraded first.

---

## 9. Integration Platform RAG Application Patterns

The crm-middleware is a natural host for RAG capabilities because it already sits between Salesforce and the EHR system, owns the data schemas, and serves a team that regularly needs to answer questions about sync behavior.

### Use case 1 — Semantic search over EHR system schema documentation

Index the EHR system API spec (endpoint descriptions, field definitions) so engineers can ask "which EHR system endpoint handles marketing submission line items?" instead of scanning a 300-page PDF.

### Use case 2 — Policy type resolution

When the EHR system returns an unrecognized policy type code, query the vector index to find the closest known `APP__Policy_Type__c` record and suggest a mapping. This turns a silent sync failure into a recoverable warning.

```python
async def resolve_policy_type(
    ehr_code: str,
    session: AsyncSession,
) -> list[dict]:
    """Find the closest known integration platform policy types for an unrecognized EHR code."""
    query = f"EHR system policy type code: {ehr_code}"
    results = await semantic_search(q=query, top_k=3, session=session)
    return [r for r in results if r["score"] > 0.75]
```

### Use case 3 — Salesforce knowledge base Q&A

Index Salesforce Knowledge articles (exported as JSON) so support engineers can ask natural-language questions about integration platform configuration without leaving the middleware admin panel.

### Integration architecture

```
[EHR API spec PDF]   ─┐
[SF Knowledge JSON]  ─┼─▶ Indexer job (async, Kubernetes CronJob) ─▶ pgvector (document_chunks)
[Platform CLAUDE.md] ─┘                                                        │
                                                                               ▼
User/Engineer ──▶ FastAPI /rag/search or /rag/ask ──▶ Embedding ──▶ pgvector query ──▶ LLM
```

**Common mistake:** Re-indexing the entire document corpus on every deploy. Use content hashes to detect changed documents and only re-embed/upsert those. Store the hash in the `metadata` JSONB column and skip chunks whose hash hasn't changed.

---

## 10. Key Concepts Summary

```
RAG Architecture
├── Indexing Pipeline
│   ├── Loaders  (PDF, JSON, Markdown, API)
│   ├── Chunking
│   │   ├── Fixed-size (simple, baseline)
│   │   ├── Recursive / semantic (better coherence)
│   │   └── Document-aware (best for structured schemas)
│   ├── Embedding Models
│   │   ├── Dense  (semantic similarity)  → bge-base, text-embedding-3-small
│   │   └── Sparse (keyword matching)     → BM25, SPLADE
│   └── Vector Stores
│       ├── pgvector      (integration platform: reuse PostgreSQL, ACID, JOIN-able)
│       ├── Pinecone      (managed, billion-scale)
│       ├── Weaviate      (built-in hybrid search + modules)
│       └── Chroma        (local dev / testing only)
│
├── ANN Index Algorithms
│   ├── HNSW  (graph-based, good at any scale, preferred for RAG)
│   └── IVFFlat (cluster-based, needs rebuild as data grows)
│
├── Query Pipeline
│   ├── Dense retrieval   (cosine distance in vector store)
│   ├── Sparse retrieval  (PostgreSQL FTS / BM25)
│   ├── Hybrid fusion     (RRF or weighted score)
│   └── LLM generation    (conditioned on retrieved chunks)
│
└── Evaluation
    ├── Retrieval  → Recall@k, Precision@k, MRR, NDCG
    └── Generation → Faithfulness, Answer Relevancy (RAGAS)
```

---

## Quiz — 20 Questions

### Questions

**1.** What problem does RAG solve that fine-tuning alone cannot?

**2.** In the RAG indexing pipeline, what is the purpose of chunk overlap?

**3.** You embed 1,000 documents with `normalize_embeddings=False` and store them in pgvector. Your cosine similarity queries return wrong results. What is the root cause?

**4.** What is the main tradeoff between HNSW and IVFFlat when documents are added incrementally over time?

**5.** Explain Reciprocal Rank Fusion (RRF) in one sentence and give the formula.

**6.** A user searches for "PDL policy" but the only relevant document says "property damage liability coverage". Which retrieval method (dense or sparse) will find it, and why?

**7.** You build an IVFFlat index with `lists = 100` on a table with only 200 rows. What happens and why?

**8.** In the integration platform context, why is pgvector preferable to a managed vector database like Pinecone for the initial RAG deployment?

**9.** What does the `ef_construction` parameter in HNSW control, and when should you increase it?

**10.** You notice your RAG system has high faithfulness scores but poor answer quality on user questions. What retrieval metric should you investigate first?

**11.** What is the difference between context precision and context recall in RAGAS?

**12.** You want to re-index integration platform documents incrementally without re-embedding unchanged chunks. What field would you add to the `document_chunks` table to support this?

**13.** A chunk retrieved from the vector store has a cosine distance of `0.85`. What does this imply about its relevance?

**14.** Why should you not use Chroma's in-memory client in a production FastAPI application?

**15.** What chunking strategy is best suited for the EHR system schema JSON that has structured object/field definitions?

**16.** Describe how you would use a single PostgreSQL query to perform hybrid search combining pgvector cosine distance with full-text search ranking.

**17.** What does `m` control in the HNSW index, and what is a practical consequence of setting it too low?

**18.** You run RAGAS and find low context recall. What does this mean, and what part of the pipeline should you adjust?

**19.** What embedding dimension would you choose for 500,000 integration platform documents stored in Cloud SQL PostgreSQL, and what is the approximate memory cost of the HNSW index?

**20.** Explain the indexing-path / query-path separation in RAG and why it matters for production systems.

---

### Answers

??? note "Reveal Answers"

    **1.** RAG solves the problem of private, domain-specific, or recently updated knowledge that was not present in the LLM's training data. Fine-tuning encodes knowledge into model weights but is expensive, slow to update, and does not give the model access to the current state of a live database. RAG retrieves fresh context at inference time from any document corpus that can be indexed, making it far cheaper to update and more auditable — the retrieved chunks serve as citations.

    **2.** Chunk overlap ensures that facts or sentences that span a chunk boundary appear in at least one complete chunk. Without overlap, a sentence that starts at the end of chunk N and finishes at the beginning of chunk N+1 would be split across two embeddings, neither of which fully captures the meaning of that sentence. A typical overlap of 10–15% of chunk size (e.g., 64 tokens for a 512-token chunk) significantly reduces boundary artifacts.

    **3.** When embeddings are not L2-normalized, the cosine distance (`<=>` in pgvector) does not equal `1 - dot_product`. For non-unit vectors, the cosine distance is affected by the magnitude of the vectors, not just their direction. Two semantically similar texts with very different magnitudes will appear far apart. Always normalize embeddings before storage using `normalize_embeddings=True` in `sentence-transformers` or `numpy.linalg.norm`.

    **4.** HNSW maintains good recall as new documents are inserted because it updates the graph incrementally — no rebuild is needed. IVFFlat's centroids are computed once at index-build time, so as new documents arrive they may fall into poorly-assigned clusters, degrading recall. IVFFlat needs a periodic `REINDEX` to recalculate centroids as the data distribution shifts. For RAG workloads where documents are added continuously, HNSW is strongly preferred.

    **5.** RRF combines multiple ranked result lists by assigning each document a score of `1 / (k + rank)` from each list and summing across lists, where `k` (typically 60) dampens the influence of top-ranked results and prevents any single list from dominating. The formula is: `RRF_score(d) = Σ 1 / (k + rank_i(d))` over all result lists `i`.

    **6.** Dense retrieval will find it, because a dense embedding model trained on a large corpus learns that "PDL" and "property damage liability" are semantically related concepts and maps them to nearby points in vector space. Sparse retrieval (BM25) would fail here because there is zero term overlap between the query "PDL policy" and the document phrase "property damage liability coverage."

    **7.** IVFFlat tries to build `lists = 100` clusters using k-means on 200 rows. With fewer than `lists * 10` rows the clusters are poorly defined — several centroids will end up with 0 or 1 document, making the index essentially useless. PostgreSQL may emit a warning or silently create a degraded index. You should always ensure `n_rows >= lists * 10` before building an IVFFlat index; for small datasets, skip the index or use HNSW.

    **8.** pgvector reuses the existing PostgreSQL infrastructure already running in the integration platform GKE cluster (Cloud SQL or self-hosted), avoiding a new managed service, new credentials, and new network egress costs. pgvector supports ACID transactions, row-level security, and JOIN operations — for example, filtering retrieved chunks by `metadata->>'object_name'` while simultaneously joining against the platform policies table. It also keeps the operational footprint minimal for a team already expert in PostgreSQL.

    **9.** `ef_construction` controls the beam width used during HNSW graph construction — how many candidate neighbors are evaluated when adding each new node to the graph. Higher values produce a better-connected graph with higher recall but increase build time. Increase it when you observe low recall on benchmark queries after building the index; a value of 128 is a good production default. The build is a one-time cost so investing in quality is almost always worthwhile.

    **10.** You should investigate **Recall@k** (retrieval recall). High faithfulness with poor answer quality means the LLM is accurately describing the chunks it received, but those chunks do not contain the information needed to answer the question. The retrieval step is failing to surface the relevant documents. Check whether relevant documents exist in the index at all, whether the chunk size is appropriate, and whether the embedding model is a good match for your domain vocabulary.

    **11.** Context precision measures whether the retrieved chunks are relevant to the question (signal-to-noise in the retrieved set). Context recall measures whether the retrieved chunks contain all the information needed to answer the question (coverage of ground-truth facts). A system can have high precision (all retrieved chunks are relevant) but low recall (missing a key fact), or high recall but low precision (the right chunks are retrieved alongside many irrelevant ones).

    **12.** You would add a `content_hash` column (e.g., `TEXT` storing the SHA-256 hex digest of the chunk content). During incremental indexing, compute the hash of each candidate chunk and query for existing rows with the same `source` and `content_hash`. Skip embedding and upsert for matching rows; only process chunks whose hash has changed or is new. This avoids redundant embedding API calls and prevents duplicate vectors.

    **13.** A cosine distance of `0.85` means the chunk is quite dissimilar to the query — cosine distance ranges from 0 (identical direction) to 2 (opposite direction), so 0.85 corresponds to a cosine similarity of `1 - 0.85 = 0.15`, which is very low. This chunk is unlikely to be relevant. In practice you should apply a threshold filter (e.g., `distance < 0.3` for 768-dim bge embeddings) and return an empty result rather than passing a low-quality chunk to the LLM, which could hallucinate.

    **14.** Chroma's default in-memory client stores all data in process memory with no persistence, meaning data is lost on restart. More critically, the in-memory client is not thread-safe and will corrupt its internal state under concurrent FastAPI requests. For production use, pgvector with your existing PostgreSQL cluster provides persistence, concurrency safety via PostgreSQL's MVCC, and ACID guarantees. Use Chroma only in local scripts and notebooks where these properties are not required.

    **15.** Document-aware chunking is best: parse the schema JSON and emit one chunk per EHR system object header and one chunk per field definition. This keeps each chunk semantically self-contained (all context about a single field in one place) and avoids splitting related fields across chunks. It also enables precise metadata tagging (`object_name`, `field_name`, `field_type`) that supports metadata filtering at query time — e.g., "only search within the Policy object."

    **16.** Run two CTEs in a single query: one selecting the top-N dense results with `ORDER BY embedding <=> $query_vec LIMIT N`, another selecting the top-N sparse results using `WHERE content @@ plainto_tsquery('english', $query_text) ORDER BY ts_rank(...) DESC LIMIT N`. Then UNION the two CTEs, apply RRF scoring in the outer query using `ROW_NUMBER() OVER (ORDER BY ...)` for each list, and return the top-k by combined RRF score. This avoids two round-trips to the database.

    **17.** `m` controls the number of bidirectional links each node maintains in the HNSW graph — it is the graph degree. Setting it too low (e.g., `m = 4`) means each node has few neighbors, so the graph is poorly connected and queries must take many hops to traverse the search space, degrading both recall and latency. Setting it too high (e.g., `m = 64`) increases memory consumption proportionally and slows index build. The practical default of `m = 16` balances these tradeoffs well for most RAG workloads.

    **18.** Low context recall means the retrieved chunks do not cover all the facts needed to answer the question — important information exists in the corpus but was not retrieved. You should adjust the retrieval stage: increase `top_k` to retrieve more chunks, lower the similarity score threshold, improve chunking (perhaps chunks are too small and split key facts), tune `ef_search` upward on the HNSW index, or switch from pure dense retrieval to hybrid search to capture keyword-matching documents that dense retrieval misses.

    **19.** For 500,000 documents at 768 dimensions, each vector costs `768 × 4 bytes = 3,072 bytes` (float32). The raw vector data is `500,000 × 3,072 ≈ 1.5 GB`. An HNSW index with `m = 16` stores approximately `m × 2 × 4 bytes` per node in graph overhead ≈ `500,000 × 128 bytes ≈ 64 MB` for graph links, plus the vector data itself. Total index memory is roughly `1.6–2.0 GB`. This is comfortably within the memory budget of a Cloud SQL Postgres instance with 4+ GB RAM, making 768-dim a practical choice over 1536-dim (which would roughly double the cost).

    **20.** The indexing path (document loading → chunking → embedding → vector store upsert) runs offline, asynchronously, and independently of user traffic — typically as a Kubernetes CronJob or an event-driven pipeline triggered on document changes. The query path (user question → embedding → vector search → LLM generation) runs synchronously on every user request and must meet a latency SLA (typically < 2 seconds). Separating them means the expensive batch work of re-indexing thousands of documents never blocks user requests, and the query path can be horizontally scaled independently from the indexing workers. In the integration platform, this maps naturally to a separate `indexer` Kubernetes Deployment with its own resource limits, decoupled from the FastAPI middleware pods.
