---
title: "Vector hóa (Local Dev - SentenceTransformer → Qdrant)"
date: 2024-01-01
weight: 8
chapter: false
pre: " <b> 5.8 </b> "
---

# Vector hóa: Tạo Embeddings & Lưu vào Vector DB

Phần này bao gồm bước vector hóa chuyển đổi text chunks thành vector embeddings và lưu trữ cho similarity search.

## Kiến trúc

### Phát triển cục bộ (Qdrant)
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

> **Note:** Trong AWS v2 architecture, vector hóa **được gộp vào ETL Lambda** — không có bước vectorize riêng biệt. Phần này chỉ bao gồm cách tiếp cận phát triển cục bộ dùng Qdrant.

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
    """Tạo embeddings và upsert lên Qdrant."""

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
        logger.info(f"Model loaded. Embedding dim: {self.embedding_size}")

    def ensure_collection(self):
        """Tạo Qdrant collection nếu chưa tồn tại."""
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
        """Lấy chunks chưa có embeddings từ PostgreSQL."""
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
                dc.title as article_title,
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
        """Tạo embeddings cho batch texts."""
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
        """Batch upsert points lên Qdrant."""
        points = []
        for chunk, vector in zip(chunks, embeddings):
            point = qmodels.PointStruct(
                id=chunk['chunk_id'],  # Dùng chunk_id làm point ID
                vector=vector,
                payload={
                    "article_id": chunk['article_id'],
                    "chunk_index": chunk['chunk_index'],
                    "content": chunk['content'],
                    "token_count": chunk['token_count'],
                    "url_hash": chunk['url_hash'],
                    "source_name": chunk['source_name'],
                    "source_domain": chunk['source_domain'],
                    "article_title": chunk['article_title'],
                }
            )
            points.append(point)

        # Upsert theo batch
        batch_size = 100
        for i in range(0, len(points), batch_size):
            batch = points[i:i + batch_size]
            self.qdrant.upsert(
                collection_name=self.collection,
                points=batch,
                wait=True,
            )
        logger.info(f"Upserted {len(points)} vectors to Qdrant")

    def mark_chunks_vectorized(self, chunk_ids: List[int]):
        """Đánh dấu chunks đã vectorized trong PostgreSQL (optional tracking)."""
        if not chunk_ids:
            return
        self.pg_cur.execute("""
            UPDATE fact_chunks
            SET embedding = '[]'::vector  -- Placeholder; real vector in Qdrant
            WHERE chunk_id = ANY(%s)
        """, (chunk_ids,))
        self.pg_conn.commit()

    def run_batch(self, limit: int = 256) -> int:
        """Xử lý một batch. Trả về số chunks đã xử lý."""
        if not self.model:
            self.load_model()

        chunks = self.get_unvectorized_chunks(limit)
        if not chunks:
            logger.info("No unvectorized chunks found")
            return 0

        texts = [c['content'] for c in chunks]
        logger.info(f"Embedding {len(texts)} chunks...")
        embeddings = self.embed_texts(texts)

        self.upsert_to_qdrant(chunks, embeddings)

        # Track trong PostgreSQL
        chunk_ids = [c['chunk_id'] for c in chunks]
        self.mark_chunks_vectorized(chunk_ids)

        return len(chunks)


def run_vectorization(limit: Optional[int] = None) -> int:
    """Entry point cho main.py --mode vectorize"""
    vec = Vectorizer()
    vec.connect_pg()
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

## Chạy Vector hóa cục bộ

```bash
# 1. Khởi động services
docker compose up -d

# 2. Chạy crawler + consumer + ETL trước
python main.py --mode crawl
python consumer/consumer.py &
python main.py --mode etl

# 3. Chạy vector hóa
python main.py --mode vectorize

# Hoặc với limit
python -c "from vectorize.vectorize import run_vectorization; run_vectorization(limit=100)"
```

## Qdrant Collection Schema

```json
{
  "collection_name": "news_chunks",
  "vectors": {
    "size": 384,
    "distance": "Cosine"
  },
  "points": [
    {
      "id": 12345,
      "vector": [0.1, -0.2, ...],
      "payload": {
        "article_id": 1001,
        "chunk_index": 0,
        "content": "Việt Nam GDP tăng trưởng 6.5%...",
        "token_count": 120,
        "url_hash": "a1b2c3...",
        "source_name": "VnExpress",
        "source_domain": "vnexpress.net",
        "article_title": "Kinh tế Việt Nam quý 3 tăng trưởng..."
      }
    }
  ]
}
```

## Chạy Vector hóa trong Main Pipeline

```bash
# Full pipeline (crawl → etl → vectorize)
python main.py --mode full

# Hoặc từng bước
python main.py --mode crawl
python main.py --mode etl
python main.py --mode vectorize
```

## AWS Production: Bedrock + pgvector

Trong production (AWS v2), vector hóa xảy ra **bên trong ETL Lambda** dùng **Amazon Bedrock Titan Embed v2**:

```python
import boto3
import json

bedrock = boto3.client("bedrock-runtime", region_name="ap-southeast-2")

def embed_with_bedrock(text: str) -> list[float]:
    """Tạo embedding dùng Amazon Titan Embed Text v2."""
    response = bedrock.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({"inputText": text})
    )
    return json.loads(response["body"].read())["embedding"]
```

**Key Differences:**

| Aspect | Local (v1) | AWS Production (v2) |
|--------|------------|---------------------|
| **Embedding Model** | BGE-small (384 dim) | Titan Embed v2 (1024 dim) |
| **Vector DB** | Qdrant (riêng) | Aurora pgvector (cùng DB) |
| **Execution** | Process Python riêng | Gộp vào ETL Lambda |
| **Index** | HNSW (Qdrant native) | HNSW (pgvector) |
| **Consistency** | Manual sync | ACID transaction với ETL |

> **Critical:** ETL và RAG API **bắt buộc dùng cùng một embedding model** (`amazon.titan-embed-text-v2:0`). Khác model = khác vector space = tìm kiếm hỏng.

## HNSW Index on pgvector (Aurora)

```sql
-- Sau khi embeddings được populate
CREATE INDEX IF NOT EXISTS idx_fact_chunks_embedding_hnsw
ON fact_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Tune search performance
SET hnsw.ef_search = 40;  -- Higher = more accurate, slower
```

## Xử lý sự cố

| Vấn đề | Giải pháp |
|--------|-----------|
| `CUDA out of memory` | Dùng CPU: `model = SentenceTransformer(model_name, device='cpu')` |
| `Qdrant connection refused` | Check `docker compose ps qdrant`, verify port 6333 |
| `Embedding dimension mismatch` | Đảm bảo `EMBEDDING_SIZE` khớp model output (384 cho BGE-small) |
| `Slow embedding` | Giảm `batch_size`, dùng GPU nếu có |
| `Collection not found` | Chạy `ensure_collection()` trước khi upsert |

## Bước tiếp theo

Sau khi vector hóa hoạt động cục bộ:
1. **Build Docker image** cho AWS: [AWS Deployment Prep](5.9-AWS-Deploy/)
2. **Deploy ETL Lambda** với Bedrock: [Lambda ETL + Embedding](5.12-Lambda-ETL/)
3. **Build RAG API**: [RAG API](5.13-RAG-API/)

---

**Tiếp theo:** [AWS Deployment Prep](5.9-AWS-Deploy/)