---
title: "Lambda ETL + Bedrock Embedding"
date: 2024-01-01
weight: 12
chapter: false
pre: " <b> 5.12 </b> "
---

# Lambda ETL + Bedrock Titan Embedding

This section covers the Lambda function that transforms raw articles into Star Schema, generates embeddings using Amazon Bedrock Titan Embed v2, and stores vectors in Aurora pgvector.

## Architecture

```
EventBridge Scheduler (02:00 UTC daily)
       │
       ▼
┌─────────────────┐
│  Lambda ETL     │  (Python 3.11, 15 min timeout, 1024 MB)
│  ─────────────  │
│  • Fetch raw    │
│  • Clean HTML   │
│  • Chunk text   │
│  • Bedrock Embed│
│  • Upsert pgvector│
└────────┬────────┘
         │
         ▼
Aurora PostgreSQL (fact_articles, fact_chunks + HNSW index)
```

## Key Design Decisions

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| **Embedding Model** | `amazon.titan-embed-text-v2:0` | 1024 dims, AWS native, serverless, consistent with RAG query |
| **Chunk Size** | ~500 tokens, 50 overlap | Balance context vs. precision |
| **Vector Index** | HNSW (m=16, ef_construction=64) | Fast approximate nearest neighbor |
| **Batch Size** | 50 articles per invocation | Stay within 15-min timeout |
| **Trigger** | EventBridge cron(0 2 * * ? *) | Daily after crawler (01:00 UTC) |

## Lambda Function Code

### File Structure
```
deploy/
├── etl/
│   ├── lambda_function.py
│   ├── requirements.txt
│   └── bedrock_utils.py
```

### bedrock_utils.py

```python
import json
import logging
import os
import boto3
from botocore.config import Config

logger = logging.getLogger(__name__)

# Bedrock client with retry config
bedrock = boto3.client(
    "bedrock-runtime",
    region_name=os.environ.get("AWS_REGION", "ap-southeast-2"),
    config=Config(
        retries={"max_attempts": 3, "mode": "adaptive"},
        read_timeout=60,
        connect_timeout=10,
    )
)

TITAN_MODEL_ID = "amazon.titan-embed-text-v2:0"
EMBEDDING_DIM = 1024


def embed_text(text: str) -> list[float]:
    """
    Generate embedding using Titan Embed v2.
    Returns list of 1024 floats.
    """
    if not text or not text.strip():
        return [0.0] * EMBEDDING_DIM

    # Truncate to model max (8192 tokens ~ 6000 words)
    # Rough approximation: 1 token ≈ 0.75 words
    max_chars = 20000
    if len(text) > max_chars:
        text = text[:max_chars]

    body = json.dumps({"inputText": text})

    try:
        response = bedrock.invoke_model(
            modelId=TITAN_MODEL_ID,
            body=body,
            contentType="application/json",
            accept="application/json"
        )
        result = json.loads(response["body"].read())
        embedding = result.get("embedding", [])

        if len(embedding) != EMBEDDING_DIM:
            logger.warning(f"Unexpected embedding dim: {len(embedding)}, expected {EMBEDDING_DIM}")
            # Pad or truncate
            if len(embedding) < EMBEDDING_DIM:
                embedding.extend([0.0] * (EMBEDDING_DIM - len(embedding)))
            else:
                embedding = embedding[:EMBEDDING_DIM]

        return embedding

    except Exception as e:
        logger.exception(f"Bedrock embed error: {e}")
        raise


def embed_batch(texts: list[str]) -> list[list[float]]:
    """Embed multiple texts sequentially (Bedrock doesn't support batch)."""
    return [embed_text(t) for t in texts]
```

### lambda_function.py

```python
import hashlib
import json
import logging
import os
import re
import sys
from datetime import datetime, timezone
from typing import List, Optional, Tuple

import psycopg2
from psycopg2.extras import RealDictCursor, execute_values

# Add layer path
sys.path.insert(0, "/opt/python")

from bedrock_utils import embed_text, EMBEDDING_DIM

logger = logging.getLogger()
logger.setLevel(logging.INFO)


# ---------- Database Connection ----------

def get_db_connection():
    """Create PostgreSQL connection."""
    return psycopg2.connect(
        host=os.environ["DB_HOST"],
        port=os.environ["DB_PORT"],
        database=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
        cursor_factory=RealDictCursor,
    )


def init_schema(conn):
    """Ensure tables and pgvector extension exist."""
    with conn.cursor() as cur:
        # Enable pgvector
        cur.execute("CREATE EXTENSION IF NOT EXISTS vector")
        
        # Star Schema tables (from warehouse.sql)
        cur.execute("""
            CREATE TABLE IF NOT EXISTS dim_source (
                source_id SERIAL PRIMARY KEY,
                source_name VARCHAR(100) NOT NULL UNIQUE,
                source_domain VARCHAR(100) NOT NULL UNIQUE,
                source_url VARCHAR(500),
                language VARCHAR(10) DEFAULT 'vi',
                country VARCHAR(10) DEFAULT 'VN',
                is_active BOOLEAN DEFAULT TRUE,
                created_at TIMESTAMPTZ DEFAULT NOW()
            )
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS dim_time (
                time_id SERIAL PRIMARY KEY,
                full_datetime TIMESTAMPTZ NOT NULL UNIQUE,
                date_day DATE NOT NULL,
                hour_24 SMALLINT NOT NULL,
                day_of_week SMALLINT NOT NULL,
                day_of_month SMALLINT NOT NULL,
                week_of_year SMALLINT NOT NULL,
                month_number SMALLINT NOT NULL,
                month_name VARCHAR(20) NOT NULL,
                quarter SMALLINT NOT NULL,
                year SMALLINT NOT NULL,
                is_weekend BOOLEAN NOT NULL,
                is_holiday BOOLEAN DEFAULT FALSE
            )
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS dim_content (
                content_id BIGSERIAL PRIMARY KEY,
                title TEXT NOT NULL,
                content_hash CHAR(64) NOT NULL UNIQUE,
                word_count INTEGER,
                char_count INTEGER,
                language VARCHAR(10) DEFAULT 'vi',
                created_at TIMESTAMPTZ DEFAULT NOW()
            )
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS dim_author (
                author_id SERIAL PRIMARY KEY,
                author_name VARCHAR(200) NOT NULL,
                author_slug VARCHAR(200),
                email VARCHAR(200),
                affiliation VARCHAR(200),
                created_at TIMESTAMPTZ DEFAULT NOW(),
                UNIQUE (author_name, affiliation)
            )
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS fact_articles (
                article_id BIGSERIAL PRIMARY KEY,
                url_hash CHAR(64) NOT NULL UNIQUE,
                source_id INTEGER NOT NULL REFERENCES dim_source(source_id),
                time_id INTEGER NOT NULL REFERENCES dim_time(time_id),
                content_id BIGINT NOT NULL REFERENCES dim_content(content_id),
                url TEXT NOT NULL,
                category VARCHAR(100),
                crawled_at TIMESTAMPTZ NOT NULL,
                embedded BOOLEAN DEFAULT FALSE,
                chunk_count INTEGER DEFAULT 0,
                created_at TIMESTAMPTZ DEFAULT NOW()
            )
        """)
        
        cur.execute("""
            CREATE TABLE IF NOT EXISTS fact_article_authors (
                article_id BIGINT NOT NULL REFERENCES fact_articles(article_id),
                author_id INTEGER NOT NULL REFERENCES dim_author(author_id),
                author_order SMALLINT DEFAULT 1,
                PRIMARY KEY (article_id, author_id)
            )
        """)
        
        # fact_chunks with pgvector
        cur.execute(f"""
            CREATE TABLE IF NOT EXISTS fact_chunks (
                chunk_id BIGSERIAL PRIMARY KEY,
                article_id BIGINT NOT NULL REFERENCES fact_articles(article_id),
                chunk_index INTEGER NOT NULL,
                content TEXT NOT NULL,
                token_count INTEGER,
                char_count INTEGER,
                embedding vector({EMBEDDING_DIM}),
                created_at TIMESTAMPTZ DEFAULT NOW(),
                UNIQUE (article_id, chunk_index)
            )
        """)
        
        # HNSW index
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_fact_chunks_embedding_hnsw
            ON fact_chunks USING hnsw (embedding vector_cosine_ops)
            WITH (m = 16, ef_construction = 64)
        """)
        
        conn.commit()


# ---------- Utility Functions ----------

def clean_html(html: str) -> str:
    """Basic HTML cleaning."""
    if not html:
        return ""
    # Remove scripts, styles
    text = re.sub(r'<script[^>]*>.*?</script>', ' ', html, flags=re.DOTALL | re.IGNORECASE)
    text = re.sub(r'<style[^>]*>.*?</style>', ' ', text, flags=re.DOTALL | re.IGNORECASE)
    # Remove tags
    text = re.sub(r'<[^>]+>', ' ', text)
    # Decode basic entities
    text = text.replace('&nbsp;', ' ').replace('&', '&').replace('<', '<').replace('>', '>')
    # Normalize whitespace
    text = re.sub(r'\s+', ' ', text)
    return text.strip()


def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
    """Split text into overlapping word chunks."""
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


def upsert_source(cur, name: str, domain: str, url: str = None) -> int:
    cur.execute("""
        INSERT INTO dim_source (source_name, source_domain, source_url)
        VALUES (%s, %s, %s)
        ON CONFLICT (source_domain) DO UPDATE SET source_name = EXCLUDED.source_name
        RETURNING source_id
    """, (name, domain, url))
    return cur.fetchone()['source_id']


def upsert_time(cur, dt: datetime) -> int:
    cur.execute("""
        INSERT INTO dim_time (
            full_datetime, date_day, hour_24, day_of_week, day_of_month,
            week_of_year, month_number, month_name, quarter, year, is_weekend
        ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON CONFLICT (full_datetime) DO NOTHING
        RETURNING time_id
    """, (
        dt, dt.date(), dt.hour, dt.isoweekday(), dt.day,
        dt.isocalendar().week, dt.month, dt.strftime('%B'),
        (dt.month - 1) // 3 + 1, dt.year, dt.weekday() >= 5
    ))
    row = cur.fetchone()
    if row:
        return row['time_id']
    cur.execute("SELECT time_id FROM dim_time WHERE full_datetime = %s", (dt,))
    return cur.fetchone()['time_id']


def upsert_content(cur, title: str, content: str) -> int:
    content_hash = hashlib.sha256(content.encode()).hexdigest()
    word_count = len(content.split())
    char_count = len(content)
    
    cur.execute("""
        INSERT INTO dim_content (title, content_hash, word_count, char_count)
        VALUES (%s, %s, %s, %s)
        ON CONFLICT (content_hash) DO NOTHING
        RETURNING content_id
    """, (title, content_hash, word_count, char_count))
    
    row = cur.fetchone()
    if row:
        return row['content_id']
    cur.execute("SELECT content_id FROM dim_content WHERE content_hash = %s", (content_hash,))
    return cur.fetchone()['content_id']


def upsert_author(cur, name: str, affiliation: str = None) -> Optional[int]:
    if not name or not name.strip():
        return None
    name = name.strip()
    cur.execute("""
        INSERT INTO dim_author (author_name, affiliation)
        VALUES (%s, %s)
        ON CONFLICT (author_name, affiliation) DO NOTHING
        RETURNING author_id
    """, (name, affiliation))
    row = cur.fetchone()
    if row:
        return row['author_id']
    cur.execute("""
        SELECT author_id FROM dim_author 
        WHERE author_name = %s AND (affiliation = %s OR (affiliation IS NULL AND %s IS NULL))
    """, (name, affiliation, affiliation))
    return cur.fetchone()['author_id']


# ---------- Main ETL Logic ----------

def process_batch(cur, articles: List[dict], batch_limit: int = 50) -> int:
    """Process a batch of raw articles. Returns count processed."""
    processed = 0
    
    for art in articles[:batch_limit]:
        try:
            url_hash = art['url_hash']
            url = art['url']
            source_name = art['source_name']
            source_domain = art['source_domain']
            title = art['title'] or ''
            content = art['content'] or ''
            category = art['category']
            author = art['author']
            published_at = art['published_at']
            crawled_at = art['crawled_at']
            raw_html = art['raw_html'] or ''
            
            # 1. Clean content
            cleaned = clean_html(raw_html or content)
            if len(cleaned) < 100:
                logger.warning(f"Skipping short article: {url}")
                cur.execute("UPDATE article_metadata SET embedded = TRUE WHERE url_hash = %s", (url_hash,))
                continue
            
            # 2. Parse published_at
            if isinstance(published_at, str):
                try:
                    pub_dt = datetime.fromisoformat(published_at.replace('Z', '+00:00'))
                except:
                    pub_dt = crawled_at or datetime.now(timezone.utc)
            elif published_at:
                pub_dt = published_at
            else:
                pub_dt = crawled_at or datetime.now(timezone.utc)
            
            # 3. Upsert dimensions
            source_id = upsert_source(cur, source_name, source_domain, url)
            time_id = upsert_time(cur, pub_dt)
            content_id = upsert_content(cur, title, cleaned)
            
            # 4. Insert fact_articles
            cur.execute("""
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
            """, (url_hash, source_id, time_id, content_id, url, category, crawled_at))
            
            article_id = cur.fetchone()['article_id']
            
            # 5. Link authors
            if author:
                for idx, author_name in enumerate([a.strip() for a in author.split(',')]):
                    author_id = upsert_author(cur, author_name)
                    if author_id:
                        cur.execute("""
                            INSERT INTO fact_article_authors (article_id, author_id, author_order)
                            VALUES (%s, %s, %s)
                            ON CONFLICT (article_id, author_id) DO NOTHING
                        """, (article_id, author_id, idx + 1))
            
            # 6. Create chunks + embeddings
            chunks = chunk_text(cleaned)
            chunk_data = []
            
            for i, chunk in enumerate(chunks):
                embedding = embed_text(chunk)
                chunk_data.append((
                    article_id, i, chunk, len(chunk.split()), len(chunk), embedding
                ))
            
            # Bulk upsert chunks
            if chunk_data:
                execute_values(cur, """
                    INSERT INTO fact_chunks (article_id, chunk_index, content, token_count, char_count, embedding)
                    VALUES %s
                    ON CONFLICT (article_id, chunk_index) DO UPDATE SET
                        content = EXCLUDED.content,
                        token_count = EXCLUDED.token_count,
                        char_count = EXCLUDED.char_count,
                        embedding = EXCLUDED.embedding
                """, chunk_data)
            
            # 7. Update article stats
            cur.execute("""
                UPDATE fact_articles 
                SET embedded = TRUE, chunk_count = %s 
                WHERE article_id = %s
            """, (len(chunks), article_id))
            
            # 8. Mark raw as processed
            cur.execute("UPDATE article_metadata SET embedded = TRUE WHERE url_hash = %s", (url_hash,))
            
            conn.commit()
            processed += 1
            logger.info(f"ETL processed: {title[:60]}... ({len(chunks)} chunks)")
            
        except Exception as e:
            conn.rollback()
            logger.exception(f"ETL failed for article {art.get('url_hash')}: {e}")
            # Continue with next article
    
    return processed


def lambda_handler(event, context):
    """Lambda entry point - triggered by EventBridge Schedule."""
    logger.info("ETL Lambda started")
    
    conn = get_db_connection()
    init_schema(conn)
    
    try:
        with conn.cursor() as cur:
            # Fetch unprocessed articles
            cur.execute("""
                SELECT 
                    id, url_hash, url, source_name, source_domain, title, content,
                    category, author, published_at, crawled_at, raw_html
                FROM article_metadata
                WHERE embedded = FALSE
                ORDER BY crawled_at ASC
                LIMIT 50
            """)
            articles = cur.fetchall()
            
            if not articles:
                logger.info("No unprocessed articles found")
                return {"statusCode": 200, "body": json.dumps({"processed": 0})}
            
            logger.info(f"Found {len(articles)} articles to process")
            
            processed = process_batch(cur, articles)
            
            logger.info(f"ETL batch complete: {processed} articles processed")
            return {
                "statusCode": 200,
                "body": json.dumps({"processed": processed})
            }
    
    except Exception as e:
        logger.exception(f"ETL Lambda error: {e}")
        return {"statusCode": 500, "body": json.dumps({"error": str(e)})}
    
    finally:
        conn.close()
```

### requirements.txt

```txt
psycopg2-binary==2.9.9
boto3==1.34.0
botocore==1.34.0
```

## Terraform Deployment

```hcl
resource "aws_lambda_function" "etl" {
  function_name = "newsrag-etl"
  description   = "ETL + Bedrock Embedding: raw -> Star Schema + pgvector"
  
  runtime       = "python3.11"
  handler       = "lambda_function.lambda_handler"
  timeout       = 900  # 15 minutes
  memory_size   = 1024
  
  filename         = "etl.zip"
  source_code_hash = filebase64sha256("etl.zip")
  
  vpc_config {
    subnet_ids         = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
    security_group_ids = [aws_security_group.lambda_sg.id]
  }
  
  role = aws_iam_role.lambda_role.arn
  
  environment {
    variables = {
      DB_HOST     = aws_rds_cluster.main.endpoint
      DB_PORT     = "5432"
      DB_NAME     = "newsrag"
      DB_USER     = var.db_user
      DB_PASSWORD = var.db_password
      AWS_REGION  = "ap-southeast-2"
    }
  }
}

# IAM permissions for Bedrock
resource "aws_iam_role_policy" "lambda_bedrock" {
  role = aws_iam_role.lambda_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["bedrock:InvokeModel"]
      Resource = "arn:aws:bedrock:ap-southeast-2::foundation-model/amazon.titan-embed-text-v2:0"
    }]
  })
}

# EventBridge Schedule (daily 02:00 UTC)
resource "aws_cloudwatch_event_rule" "etl_schedule" {
  name                = "newsrag-etl-rule"
  schedule_expression = "cron(0 2 * * ? *)"
}

resource "aws_cloudwatch_event_target" "etl_target" {
  rule      = aws_cloudwatch_event_rule.etl_schedule.name
  arn       = aws_lambda_function.etl.arn
  role_arn  = aws_iam_role.lambda_role.arn
}

resource "aws_lambda_permission" "etl_eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.etl.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.etl_schedule.arn
}
```

## Testing the ETL Lambda

### 1. Invoke Manually

```bash
aws lambda invoke \
  --function-name newsrag-etl \
  --payload '{}' \
  response.json

cat response.json
```

### 2. Check Logs

```bash
aws logs tail /aws/lambda/newsrag-etl --follow
```

### 3. Verify in Database

```bash
ENDPOINT=$(terraform output -raw rds_endpoint)
psql "postgresql://postgres:password@${ENDPOINT}:5432/newsrag" -c "
SELECT 
    fa.article_id,
    ds.source_name,
    fa.chunk_count,
    fa.embedded,
    fc.chunk_id,
    fc.content,
    fc.embedding IS NOT NULL as has_embedding
FROM fact_articles fa
JOIN dim_source ds ON fa.source_id = ds.source_id
LEFT JOIN fact_chunks fc ON fa.article_id = fc.article_id
ORDER BY fa.created_at DESC
LIMIT 10;
"
```

## Monitoring & Optimization

### CloudWatch Metrics

| Metric | Target | Alert If |
|--------|--------|----------|
| Duration | < 600s (p95) | > 800s |
| Errors | 0 | > 0 |
| Invocations | 1/day | = 0 for 2 days |
| Bedrock Invocations | ~1500/day | Unexpected spike |

### Cost Optimization

- **Bedrock Titan Embed v2**: ~$0.0001 per 1K tokens
- 50 articles × 10 chunks × 500 tokens = 250K tokens/day = ~$0.025/day = ~$0.75/month
- Lambda: 15 min × 1 GB × 1/day = negligible
- **Total ETL cost**: ~$1-2/month

### Performance Tips

1. **Connection pooling**: Use RDS Proxy for high-frequency invocations
2. **Batch embeddings**: Process chunks in batches (though Bedrock is sequential)
3. **Retry logic**: Exponential backoff for Bedrock throttling
4. **Monitor HNSW index**: `ef_search` parameter at query time

## Next Steps

After ETL + Embedding works:
1. **RAG API** - Lambda + API Gateway for querying: [RAG API](5.13-RAG-API/)
2. **Frontend** - Next.js dashboard: [Frontend Integration](5.14-Frontend/)

---

**Next:** [RAG API (Lambda + API Gateway)](5.13-RAG-API/)