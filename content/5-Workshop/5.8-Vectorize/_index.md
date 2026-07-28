---
title: "Vectorization (Local Dev - SentenceTransformer → Qdrant)"
date: 2024-01-01
weight: 8
chapter: false
pre: " <b> 5.8 </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# Vectorization: Generating Embeddings & Storing in Vector DB

This section covers the vectorization step that converts text chunks into vector embeddings and stores them for similarity search.

## Architecture

### Local Development (Qdrant)
```
fact_chunks (PostgreSQL)
       │
       ▼
SentenceTransformer (local model)
       │
       ▼
Qdrant Vector DB (collection: news_chunks)
```

### AWS Production (Aurora pgvector)
```
fact_chunks (Aurora PostgreSQL)
       │
       ▼
Lambda ETL + Bedrock Titan Embed v2
       │
       ▼
fact_chunks.embedding (pgvector + HNSW)
```

> **Note:** In AWS v2 architecture, vectorization is **merged into the ETL Lambda** — no separate vectorize step. This section covers the local development approach using Qdrant.

## Vectorization Implementation (vectorize/vectorize.py)

```python
import os
import logging
from typing import List, Optional

import psycopg2
from psycopg2.extras import RealDictCursor
from qdrant_client import QdrantClient
from qdrant_client.http import models as qmodels
from sentence_transformers import SentenceTransformer

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class Vectorizer:
    """Generate embeddings and upsert to Qdrant."""

    def __init__(self):
        # PostgreSQL connection
        self.pg_dsn = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
        self.pg_conn = None
        self.pg_cur = None

        # Qdrant client
        self.qdrant = QdrantClient(
            host=os.getenv("QDRANT_HOST", "localhost"),
            port=int(os.getenv("QDRANT_PORT", "6333")),
            api_key=os.getenv("QDRANT_API_KEY") or None,
        )
        self.collection = os.getenv("QDRANT_COLLECTION_NAME", "news_chunks")

        # Embedding model
        self.model_name = os.getenv("EMBEDDING_MODEL", "BAAI/bge-small-en-v1.5")
        self.embedding_size = int(os.getenv("EMBEDDING_SIZE", "384"))
        self.model = None

    def connect_pg(self):
        self.pg_conn = psycopg2.connect(self.pg_dsn, cursor_factory=RealDictCursor)
        self.pg_cur = self.pg_conn.cursor()

    def close_pg(self):
        if self.pg_cur: self.pg_cur.close()
        if self.pg_conn: self.pg_conn.close()

    def load_model(self):
        """Load SentenceTransformer model (downloads on first run)."""
        logger.info(f"Loading embedding model: {self.model_name}")
        self.model = SentenceTransformer(self.model_name)
        actual_dim = self.model.get_sentence_embedding_dimension()
        if actual_dim != self.embedding_size:
            logger.warning(f"Model dim {actual_dim} != configured {self.embedding_size}")
            self.embedding_size = actual_dim
        logger.info(f"Model loaded. Embedding dimension: {self.embedding_size}")

    def ensure_collection(self):
        """Create Qdrant collection if not exists."""
        collections = self.qdrant.get_collections().collections
        names = [c.name for c in collections]

        if self.collection not in names:
            logger.info(f"Creating Qdrant collection: {self.collection}")
            self.qdrant.create_collection(
                collection_name=self.collection,
                vectors_config=qmodels.VectorParams(
                    size=self.embedding_size,
                    distance=qmodels.Distance.COSINE,
                ),
                optimizers_config=qmodels.OptimizersConfigDiff(
                    default_segment_number=2,
                    indexing_threshold=10000,
                ),
            )
        else:
            logger.info(f"Collection {self.collection} already exists")

    def get_unvectorized_chunks(self, limit: int = 256) -> List[dict]:
        """Fetch chunks without embeddings from PostgreSQL."""
        self.pg_cur.execute("""
            SELECT
                fc.chunk_id,
                fc.article_id,
                fc.chunk_index,
                fc.content,
                fc.token_count,
                fa.url_hash,
                fa.source_id,
                fa.time_id,
                dc.title,
                ds.source_name,
                ds.source_domain
            FROM fact_chunks fc
            JOIN fact_articles fa ON fc.article_id = fa.article_id
            JOIN dim_content dc ON fa.content_id = dc.content_id
            JOIN dim_source ds ON fa.source_id = ds.source_id
            WHERE fc.embedding IS NULL
            ORDER BY fc.chunk_id
            LIMIT %s
        """, (limit,))
        return self.pg_cur.fetchall()

    def embed_texts(self, texts: List[str]) -> List[List[float]]:
        """Generate embeddings for a batch of texts."""
        if not texts:
            return []
        embeddings = self.model.encode(
            texts,
            batch_size=32,
            show_progress_bar=True,
            convert_to_numpy=True,
            normalize_embeddings=True,  # For cosine similarity
        )
        return embeddings.tolist()

    def upsert_to_qdrant(self, chunks: List[dict], embeddings: List[List[float]]):
        """Upsert points to Qdrant."""
        points = []
        for chunk, embedding in zip(chunks, embeddings):
            point_id = chunk["chunk_id"]
            payload = {
                "article_id": chunk["article_id"],
                "chunk_index": chunk["chunk_index"],
                "content": chunk["content"],
                "token_count": chunk["token_count"],
                "url_hash": chunk["url_hash"],
                "source_name": chunk["source_name"],
                "source_domain": chunk["source_domain"],
                "title": chunk["title"],
            }
            points.append(qmodels.PointStruct(
                id=point_id,
                vector=embedding,
                payload=payload
            ))

        self.qdrant.upsert(
            collection_name=self.collection,
            points=points,
            wait=True
        )
        logger.info(f"Upserted {len(points)} vectors to Qdrant")

    def mark_embedded(self, chunk_ids: List[int]):
        """Mark chunks as embedded in PostgreSQL."""
        if not chunk_ids:
            return
        placeholders = ",".join(["%s"] * len(chunk_ids))
        self.pg_cur.execute(
            f"UPDATE fact_chunks SET embedding = TRUE WHERE chunk_id IN ({placeholders})",
            tuple(chunk_ids)
        )
        self.pg_conn.commit()

    def run_batch(self, limit: int = 256) -> int:
        """Process one batch of chunks. Returns count processed."""
        chunks = self.get_unvectorized_chunks(limit)
        if not chunks:
            return 0

        texts = [c["content"] for c in chunks]
        embeddings = self.embed_texts(texts)

        self.upsert_to_qdrant(chunks, embeddings)
        self.mark_embedded([c["chunk_id"] for c in chunks])

        logger.info(f"Vectorized batch: {len(chunks)} chunks")
        return len(chunks)


def run_vectorization(limit: Optional[int] = None) -> int:
    """Entry point for main.py --mode vectorize"""
    vec = Vectorizer()
    vec.connect_pg()
    vec.load_model()
    vec.ensure_collection()

    try:
        total = 0
        batch_size = 256
        while True:
            processed = vec.run_batch(batch_size)
            total += processed
            if processed == 0 or (limit and total >= limit):
                break
        logger.info(f"Vectorization complete. Total chunks: {total}")
        return total
    finally:
        vec.close_pg()


if __name__ == "__main__":
    run_vectorization()
```

## Running Vectorization Locally

```bash
# 1. Start services
docker compose up -d

# 2. Run crawler + consumer + ETL first
python main.py --mode crawl
python consumer/consumer.py &
python main.py --mode etl

# 3. Run vectorization
python main.py --mode vectorize

# Or with limit
python -c "from vectorize.vectorize import run_vectorization; run_vectorization(limit=100)"
```

## Verifying in Qdrant

```bash
# Check collection info
curl http://localhost:6333/collections/news_chunks

# Search test
curl -X POST http://localhost:6333/collections/news_chunks/points/search \
  -H "Content-Type: application/json" \
  -d '{
    "vector": [0.1, 0.2, ...],  # 384-dim vector
    "limit": 5,
    "with_payload": true
  }'

# Or use Qdrant UI
open http://localhost:6333/dashboard
```

## Qdrant Collection Configuration

| Setting | Value | Reason |
|---------|-------|--------|
| Distance | `COSINE` | Normalized embeddings, cosine = dot product |
| Vector Size | 384 (BGE-small) / 1024 (Titan v2) | Match embedding model |
| HNSW `m` | 16 | Balance memory vs. recall |
| HNSW `ef_construct` | 64 | Build-time recall/quality tradeoff |

## Running Vectorization in Main Pipeline

```bash
# Full pipeline (crawl → etl → vectorize)
python main.py --mode full

# Or individual steps
python main.py --mode crawl
python main.py --mode etl
python main.py --mode vectorize
```

## AWS Production: ETL + Embedding Merged

In the AWS deployment, the vectorize step is **not separate**. The Lambda ETL function:
1. Fetches unprocessed articles
2. Cleans HTML, chunks text (~500 tokens)
3. Calls `bedrock.invoke_model` for each chunk
4. Inserts directly into `fact_chunks.embedding` (pgvector)

See [Lambda ETL + Embedding](5.12-Lambda-ETL/) for the production implementation.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Qdrant connection refused` | Ensure Qdrant container is running: `docker compose ps qdrant` |
| `Model download fails` | Check internet access; model caches at `~/.cache/huggingface/` |
| `Dimension mismatch` | Verify `EMBEDDING_SIZE` matches model output dimension |
| `Out of memory` | Reduce `batch_size` in `model.encode()`, or use smaller model |
| `Slow embedding` | First run downloads model (~100MB); subsequent runs use cache |

---

**Next:** [AWS Deployment Prep](5.9-AWS-Deploy/)