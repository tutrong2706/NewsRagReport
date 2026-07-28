---
title: "ETL & Star Schema"
date: 2026-07-28
weight: 7
chapter: false
pre: " <b> 5.7 </b> "
---

# ETL & Star Schema Transformation

This section covers the ETL process that transforms raw articles from `article_metadata` into a Star Schema optimized for analytics and RAG retrieval.

## Star Schema Design

```
                    ┌─────────────────┐
                    │   fact_articles │ ◄─── Core fact table
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  dim_source   │    │   dim_time    │    │  dim_content  │
└───────────────┘    └───────────────┘    └───────────────┘
        │                    │                    │
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  dim_author   │    │ fact_article_ │    │  fact_chunks  │ ◄─── For RAG
│               │    │   _authors    │    │  (vectors)    │
└───────────────┘    └───────────────┘    └───────────────┘
```

## Dimension Tables

### dim_source — News Sources
```sql
CREATE TABLE dim_source (
    source_id       SERIAL PRIMARY KEY,
    source_name     VARCHAR(100) NOT NULL UNIQUE,
    source_domain   VARCHAR(100) NOT NULL UNIQUE,
    source_url      VARCHAR(500),
    language        VARCHAR(10) DEFAULT 'vi',
    country         VARCHAR(10) DEFAULT 'VN',
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### dim_time — Time Dimension
```sql
CREATE TABLE dim_time (
    time_id         SERIAL PRIMARY KEY,
    full_datetime   TIMESTAMPTZ NOT NULL UNIQUE,
    date_day        DATE NOT NULL,
    hour_24         SMALLINT NOT NULL,      -- 0-23
    day_of_week     SMALLINT NOT NULL,      -- 1=Mon, 7=Sun
    day_of_month    SMALLINT NOT NULL,
    week_of_year    SMALLINT NOT NULL,
    month_number    SMALLINT NOT NULL,
    month_name      VARCHAR(20) NOT NULL,
    quarter         SMALLINT NOT NULL,
    year            SMALLINT NOT NULL,
    is_weekend      BOOLEAN NOT NULL,
    is_holiday      BOOLEAN DEFAULT FALSE   -- VN holidays (extend as needed)
);
```

### dim_content — Article Content
```sql
CREATE TABLE dim_content (
    content_id      BIGSERIAL PRIMARY KEY,
    title           TEXT NOT NULL,
    content_hash    CHAR(64) NOT NULL UNIQUE,  -- SHA256 for dedup
    word_count      INTEGER,
    char_count      INTEGER,
    language        VARCHAR(10) DEFAULT 'vi',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### dim_author — Authors
```sql
CREATE TABLE dim_author (
    author_id       SERIAL PRIMARY KEY,
    author_name     VARCHAR(200) NOT NULL,
    author_slug     VARCHAR(200),              -- URL-friendly
    email           VARCHAR(200),
    affiliation     VARCHAR(200),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (author_name, affiliation)
);
```

## Fact Tables

### fact_articles — Core Fact Table
```sql
CREATE TABLE fact_articles (
    article_id          BIGSERIAL PRIMARY KEY,
    url_hash            CHAR(64) NOT NULL UNIQUE,
    source_id           INTEGER NOT NULL REFERENCES dim_source(source_id),
    time_id             INTEGER NOT NULL REFERENCES dim_time(time_id),
    content_id          BIGINT NOT NULL REFERENCES dim_content(content_id),
    url                 TEXT NOT NULL,
    category            VARCHAR(100),
    crawled_at          TIMESTAMPTZ NOT NULL,
    embedded            BOOLEAN DEFAULT FALSE,      -- ETL → vectorization flag
    chunk_count         INTEGER DEFAULT 0,         -- Populated after vectorize
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_fact_articles_source_time ON fact_articles(source_id, time_id);
CREATE INDEX idx_fact_articles_embedded ON fact_articles(embedded) WHERE embedded = FALSE;
```

### fact_article_authors — Many-to-Many Bridge
```sql
CREATE TABLE fact_article_authors (
    article_id      BIGINT NOT NULL REFERENCES fact_articles(article_id),
    author_id       INTEGER NOT NULL REFERENCES dim_author(author_id),
    author_order    SMALLINT DEFAULT 1,           -- 1=primary, 2=secondary...
    PRIMARY KEY (article_id, author_id)
);
```

### fact_chunks — RAG Vector Chunks (with pgvector)
```sql
CREATE TABLE fact_chunks (
    chunk_id        BIGSERIAL PRIMARY KEY,
    article_id      BIGINT NOT NULL REFERENCES fact_articles(article_id),
    chunk_index     INTEGER NOT NULL,
    content         TEXT NOT NULL,
    token_count     INTEGER,
    char_count      INTEGER,
    embedding       vector(1024),                 -- Titan Embed v2 = 1024 dims
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (article_id, chunk_index)
);

-- HNSW Index for fast similarity search
CREATE INDEX idx_fact_chunks_embedding_hnsw
ON fact_chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

## ETL Process (etl/etl_warehouse.py)

```python
import hashlib
import re
import os
import logging
from datetime import datetime
from typing import Optional, List, Tuple

import psycopg2
from psycopg2.extras import RealDictCursor, execute_values

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class ETLWarehouse:
    """Transform raw articles into Star Schema."""

    def __init__(self, dsn: str):
        self.dsn = dsn
        self.conn = None
        self.cur = None

    def connect(self):
        self.conn = psycopg2.connect(self.dsn, cursor_factory=RealDictCursor)
        self.cur = self.conn.cursor()

    def close(self):
        if self.cur: self.cur.close()
        if self.conn: self.conn.close()

    # ---------- Dimension Upserts ----------

    def upsert_source(self, name: str, domain: str, url: str = None) -> int:
        self.cur.execute("""
            INSERT INTO dim_source (source_name, source_domain, source_url)
            VALUES (%s, %s, %s)
            ON CONFLICT (source_domain) DO UPDATE SET source_name = EXCLUDED.source_name
            RETURNING source_id
        """, (name, domain, url))
        return self.cur.fetchone()['source_id']

    def upsert_time(self, dt: datetime) -> int:
        self.cur.execute("""
            INSERT INTO dim_time (
                full_datetime, date_day, hour_24, day_of_week, day_of_month,
                week_of_year, month_number, month_name, quarter, year,
                is_weekend
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (full_datetime) DO NOTHING
            RETURNING time_id
        """, (
            dt, dt.date(), dt.hour, dt.isoweekday(), dt.day,
            dt.isocalendar().week, dt.month, dt.strftime('%B'),
            (dt.month - 1) // 3 + 1, dt.year, dt.weekday() >= 5
        ))
        row = self.cur.fetchone()
        if row:
            return row['time_id']
        # Already exists
        self.cur.execute("SELECT time_id FROM dim_time WHERE full_datetime = %s", (dt,))
        return self.cur.fetchone()['time_id']

    def upsert_content(self, title: str, content: str) -> int:
        content_hash = hashlib.sha256(content.encode()).hexdigest()
        word_count = len(content.split())
        char_count = len(content)

        self.cur.execute("""
            INSERT INTO dim_content (title, content_hash, word_count, char_count)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (content_hash) DO NOTHING
            RETURNING content_id
        """, (title, content_hash, word_count, char_count))

        row = self.cur.fetchone()
        if row:
            return row['content_id']

        self.cur.execute("SELECT content_id FROM dim_content WHERE content_hash = %s", (content_hash,))
        return self.cur.fetchone()['content_id']

    def upsert_author(self, name: str, affiliation: str = None) -> Optional[int]:
        if not name or not name.strip():
            return None
        name = name.strip()
        self.cur.execute("""
            INSERT INTO dim_author (author_name, affiliation)
            VALUES (%s, %s)
            ON CONFLICT (author_name, affiliation) DO NOTHING
            RETURNING author_id
        """, (name, affiliation))
        row = self.cur.fetchone()
        if row:
            return row['author_id']
        self.cur.execute(
            "SELECT author_id FROM dim_author WHERE author_name = %s AND (affiliation = %s OR (affiliation IS NULL AND %s IS NULL))",
            (name, affiliation, affiliation)
        )
        return self.cur.fetchone()['author_id']

    # ---------- Chunking ----------

    def chunk_text(self, text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
        """Split text into overlapping chunks of ~chunk_size words."""
        words = text.split()
        if len(words) <= chunk_size:
            return [text]

        chunks = []
        step = chunk_size - overlap
        for i in range(0, len(words), step):
            chunk = " ".join(words[i:i + chunk_size])
            if len(chunk.strip()) > 50:  # Skip tiny chunks
                chunks.append(chunk)
        return chunks

    # ---------- Main ETL ----------

    def clean_html(self, html: str) -> str:
        """Basic HTML cleaning."""
        if not html:
            return ""
        # Remove scripts, styles
        text = re.sub(r'<script[^>]*>.*?</script>', ' ', html, flags=re.DOTALL | re.IGNORECASE)
        text = re.sub(r'<style[^>]*>.*?</style>', ' ', text, flags=re.DOTALL | re.IGNORECASE)
        # Remove tags
        text = re.sub(r'<[^>]+>', ' ', text)
        # Decode entities (basic)
        text = text.replace('&nbsp;', ' ').replace('&', '&').replace('<', '<').replace('>', '>')
        # Normalize whitespace
        text = re.sub(r'\s+', ' ', text)
        return text.strip()

    def process_batch(self, limit: int = 50) -> int:
        """Process unprocessed articles. Returns count processed."""
        self.cur.execute("""
            SELECT id, url_hash, url, source_name, source_domain, title, content,
                   category, author, published_at, crawled_at, raw_html
            FROM article_metadata
            WHERE embedded = FALSE
            ORDER BY crawled_at ASC
            LIMIT %s
        """, (limit,))

        articles = self.cur.fetchall()
        if not articles:
            return 0

        processed = 0
        for art in articles:
            try:
                # 1. Clean content
                raw_html = art['raw_html'] or art['content'] or ''
                cleaned = self.clean_html(raw_html)
                if not cleaned or len(cleaned) < 100:
                    logger.warning(f"Skipping short article: {art['url']}")
                    self.cur.execute("UPDATE article_metadata SET embedded = TRUE WHERE id = %s", (art['id'],))
                    self.conn.commit()
                    continue

                # 2. Parse published_at
                pub_dt = art['published_at']
                if isinstance(pub_dt, str):
                    try:
                        pub_dt = datetime.fromisoformat(pub_dt.replace('Z', '+00:00'))
                    except:
                        pub_dt = art['crawled_at'] or datetime.utcnow()

                # 3. Upsert dimensions
                source_id = self.upsert_source(art['source_name'], art['source_domain'], art['url'])
                time_id = self.upsert_time(pub_dt)
                content_id = self.upsert_content(art['title'], cleaned)

                # 4. Insert fact_articles
                self.cur.execute("""
                    INSERT INTO fact_articles (
                        url_hash, source_id, time_id, content_id, url, category, crawled_at
                    ) VALUES (%s, %s, %s, %s, %s, %s, %s)
                    ON CONFLICT (url_hash) DO UPDATE SET
                        source_id = EXCLUDED.source_id,
                        time_id = EXCLUDED.time_id,
                        content_id = EXCLUDED.content_id,
                        category = EXCLUDED.category,
                        crawled_at = EXCLUDED.crawled_at
                    RETURNING article_id
                """, (art['url_hash'], source_id, time_id, content_id, art['url'], art['category'], art['crawled_at']))

                article_id = self.cur.fetchone()['article_id']

                # 5. Link authors
                if art['author']:
                    for idx, author_name in enumerate([a.strip() for a in art['author'].split(',')]):
                        author_id = self.upsert_author(author_name)
                        if author_id:
                            self.cur.execute("""
                                INSERT INTO fact_article_authors (article_id, author_id, author_order)
                                VALUES (%s, %s, %s)
                                ON CONFLICT (article_id, author_id) DO NOTHING
                            """, (article_id, author_id, idx + 1))

                # 6. Create chunks (embedding done later in vectorize step)
                chunks = self.chunk_text(cleaned)
                for i, chunk in enumerate(chunks):
                    self.cur.execute("""
                        INSERT INTO fact_chunks (article_id, chunk_index, content, token_count, char_count)
                        VALUES (%s, %s, %s, %s, %s)
                        ON CONFLICT (article_id, chunk_index) DO UPDATE SET
                            content = EXCLUDED.content,
                            token_count = EXCLUDED.token_count,
                            char_count = EXCLUDED.char_count
                    """, (article_id, i, chunk, len(chunk.split()), len(chunk)))

                # 7. Mark raw article as processed
                self.cur.execute("UPDATE article_metadata SET embedded = TRUE WHERE id = %s", (art['id'],))

                self.conn.commit()
                processed += 1
                logger.info(f"ETL processed: {art['title'][:60]}... ({len(chunks)} chunks)")

            except Exception as e:
                self.conn.rollback()
                logger.exception(f"ETL failed for article {art['id']}: {e}")

        return processed


def run_etl_warehouse(limit: Optional[int] = None) -> int:
    """Entry point for main.py --mode etl"""
    dsn = f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    etl = ETLWarehouse(dsn)
    etl.connect()
    try:
        total = 0
        batch_size = 50
        while True:
            processed = etl.process_batch(batch_size)
            total += processed
            if processed == 0 or (limit and total >= limit):
                break
        logger.info(f"ETL completed. Total articles processed: {total}")
        return total
    finally:
        etl.close()


if __name__ == "__main__":
    run_etl_warehouse()
```

## Running ETL Locally

```bash
# After crawler + consumer have populated article_metadata
python main.py --mode etl

# Or with limit
python -c "from etl.etl_warehouse import run_etl_warehouse; run_etl_warehouse(limit=100)"
```

## Verifying Star Schema

```sql
-- Check dimension counts
SELECT 'dim_source' AS table, COUNT(*) FROM dim_source
UNION ALL SELECT 'dim_time', COUNT(*) FROM dim_time
UNION ALL SELECT 'dim_content', COUNT(*) FROM dim_content
UNION ALL SELECT 'dim_author', COUNT(*) FROM dim_author;

-- Check fact counts
SELECT 'fact_articles' AS table, COUNT(*) FROM fact_articles
UNION ALL SELECT 'fact_article_authors', COUNT(*) FROM fact_article_authors
UNION ALL SELECT 'fact_chunks', COUNT(*) FROM fact_chunks;

-- Sample join query
SELECT
    a.article_id,
    s.source_name,
    t.date_day,
    c.title,
    a.chunk_count
FROM fact_articles a
JOIN dim_source s ON a.source_id = s.source_id
JOIN dim_time t ON a.time_id = t.time_id
JOIN dim_content c ON a.content_id = c.content_id
WHERE a.embedded = FALSE
ORDER BY a.created_at DESC
LIMIT 10;
```

## AWS Lambda Version (Production)

For AWS, the ETL runs as a **Lambda function** triggered by EventBridge (02:00 UTC), using **Bedrock Titan Embed v2** for embeddings directly in the same function. See [Lambda ETL + Embedding](5.12-Lambda-ETL/).

Key differences:
- **Embedding**: Calls `bedrock-runtime.invoke_model` instead of local SentenceTransformer
- **Vector DB**: Writes to Aurora `fact_chunks.embedding` (pgvector) instead of Qdrant
- **Secrets**: Reads DB credentials from AWS Secrets Manager
- **Timeout**: 15 minutes (processes ~50 articles per invocation)

---

**Next:** [Vectorization](5.8-Vectorize/)