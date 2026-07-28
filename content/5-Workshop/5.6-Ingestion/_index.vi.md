---
title: "Nhập dữ liệu (Kafka Consumer → PostgreSQL)"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.6 </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Thông tin dưới đây chỉ mang tính tham khảo. Vui lòng **không sao chép y nguyên** cho báo cáo của bạn, bao gồm cả cảnh báo này.
{{% /notice %}}

# Nhập dữ liệu: Kafka Consumer → PostgreSQL

Phần này bao gồm consumer service đọc bài viết thô từ Kafka và lưu vào PostgreSQL.

## Kiến trúc

```
Kafka (news_raw topic)
       │
       ▼
┌──────────────────┐
│  Lambda Consumer │  (AWS: Lambda triggered by SQS)
│  or              │
│  Python Process  │  (Local: consumer/consumer.py)
└──────────────────┘
       │
       ▼
PostgreSQL (article_metadata table)
```

## Database Schema (Raw Table)

```sql
-- Part of database/warehouse.sql
CREATE TABLE IF NOT EXISTS article_metadata (
    id              BIGSERIAL PRIMARY KEY,
    url_hash        CHAR(64) UNIQUE NOT NULL,    -- SHA256 hex
    url             TEXT NOT NULL,
    source_name     VARCHAR(100),
    source_domain   VARCHAR(100),
    title           TEXT,
    content         TEXT,
    category        VARCHAR(100),
    author          VARCHAR(200),
    published_at    TIMESTAMPTZ,
    crawled_at      TIMESTAMPTZ,
    raw_html        TEXT,
    embedded        BOOLEAN DEFAULT FALSE,        -- ETL status flag
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Index cho deduplication lookups
CREATE INDEX IF NOT EXISTS idx_article_metadata_url_hash ON article_metadata(url_hash);
CREATE INDEX IF NOT EXISTS idx_article_metadata_published_at ON article_metadata(published_at DESC);
```

## Consumer Implementation (consumer/consumer.py)

```python
import hashlib
import json
import logging
import os
import signal
import sys
import time
from contextlib import contextmanager
from typing import Optional

import psycopg2
from psycopg2.extras import RealDictCursor
from kafka import KafkaConsumer
from kafka.errors import KafkaError

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger(__name__)


class NewsConsumer:
    """Kafka consumer ghi bài viết vào PostgreSQL với deduplication."""

    def __init__(self):
        self.running = True
        self.consumer = None
        self.conn = None
        self.cur = None

        # Graceful shutdown
        signal.signal(signal.SIGINT, self._signal_handler)
        signal.signal(signal.SIGTERM, self._signal_handler)

    def _signal_handler(self, signum, frame):
        logger.info(f"Received signal {signum}, shutting down...")
        self.running = False

    def _connect_db(self):
        """Kết nối PostgreSQL với retry."""
        max_retries = 5
        for attempt in range(max_retries):
            try:
                self.conn = psycopg2.connect(
                    host=os.getenv("DB_HOST", "localhost"),
                    port=os.getenv("DB_PORT", "5432"),
                    database=os.getenv("DB_NAME", "newsrag"),
                    user=os.getenv("DB_USER", "postgres"),
                    password=os.getenv("DB_PASSWORD", "postgres"),
                    cursor_factory=RealDictCursor,
                )
                self.conn.autocommit = False
                self.cur = self.conn.cursor()
                logger.info("Connected to PostgreSQL")
                return
            except Exception as e:
                logger.warning(f"DB connection attempt {attempt + 1}/{max_retries} failed: {e}")
                if attempt == max_retries - 1:
                    raise
                time.sleep(2 ** attempt)

    def _connect_kafka(self):
        """Tạo Kafka consumer."""
        self.consumer = KafkaConsumer(
            os.getenv("KAFKA_TOPIC_NEWS", "news_raw"),
            bootstrap_servers=os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092"),
            group_id="newsrag-consumer-group",
            value_deserializer=lambda m: json.loads(m.decode("utf-8")),
            key_deserializer=lambda k: k.decode("utf-8") if k else None,
            auto_offset_reset="earliest",
            enable_auto_commit=True,
            auto_commit_interval_ms=5000,
            max_poll_records=100,
            session_timeout_ms=30000,
            heartbeat_interval_ms=10000,
        )
        logger.info("Kafka consumer connected")

    def _insert_article(self, article: dict) -> bool:
        """
        Insert bài viết với ON CONFLICT DO NOTHING cho deduplication.
        Returns True nếu inserted, False nếu duplicate.
        """
        url_hash = article.get("url_hash") or hashlib.sha256(article["url"].encode()).hexdigest()

        sql = """
            INSERT INTO article_metadata (
                url_hash, url, source_name, source_domain, title, content,
                category, author, published_at, crawled_at, raw_html
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (url_hash) DO NOTHING
            RETURNING id
        """
        try:
            self.cur.execute(sql, (
                url_hash,
                article["url"],
                article.get("source_name"),
                article.get("source_domain"),
                article.get("title"),
                article.get("content"),
                article.get("category"),
                article.get("author"),
                article.get("published_at"),
                article.get("crawled_at"),
                article.get("raw_html", "")[:50000],  # Limit size
            ))
            self.conn.commit()
            return self.cur.rowcount > 0
        except Exception as e:
            self.conn.rollback()
            logger.error(f"Insert failed for {article.get('url')}: {e}")
            return False

    def process_message(self, message) -> bool:
        """Xử lý một Kafka message."""
        article = message.value
        if not article or not article.get("url"):
            logger.warning("Empty or invalid message")
            return False

        inserted = self._insert_article(article)
        if inserted:
            logger.info(f"Inserted: {article['url'][:80]}...")
        else:
            logger.debug(f"Duplicate skipped: {article['url'][:80]}...")
        return inserted

    def run(self):
        """Main consumer loop."""
        self._connect_db()
        self._connect_kafka()

        logger.info("Starting consumer loop...")
        inserted_count = 0

        try:
            while self.running:
                # Poll for messages (timeout 1s for responsive shutdown)
                records = self.consumer.poll(timeout_ms=1000)

                for topic_partition, messages in records.items():
                    for message in messages:
                        if not self.running:
                            break
                        if self.process_message(message):
                            inserted_count += 1

                        # Log progress periodically
                        if inserted_count % 50 == 0 and inserted_count > 0:
                            logger.info(f"Processed {inserted_count} new articles")

        except KeyboardInterrupt:
            logger.info("Interrupted by user")
        except Exception as e:
            logger.exception(f"Consumer error: {e}")
        finally:
            self.shutdown()

        logger.info(f"Consumer finished. Total inserted: {inserted_count}")

    def shutdown(self):
        """Clean up connections."""
        if self.consumer:
            self.consumer.close()
            logger.info("Kafka consumer closed")
        if self.cur:
            self.cur.close()
        if self.conn:
            self.conn.close()
            logger.info("Database connection closed")


def start_processing():
    """Entry point cho main.py --mode consumer"""
    consumer = NewsConsumer()
    consumer.run()


if __name__ == "__main__":
    start_processing()
```

## Chạy cục bộ

```bash
# Terminal 1: Khởi động services
docker compose up -d

# Terminal 2: Chạy consumer (chạy liên tục)
python consumer/consumer.py

# Terminal 3: Chạy crawler để produce messages
python main.py --mode crawl
```

## Xác minh dữ liệu trong PostgreSQL

```bash
# Kết nối DB
psql -h localhost -U postgres -d newsrag

# Kiểm tra bài đã insert
SELECT source_name, title, published_at, embedded
FROM article_metadata
ORDER BY created_at DESC
LIMIT 10;

# Đếm theo source
SELECT source_name, COUNT(*) as count
FROM article_metadata
GROUP BY source_name;

# Kiểm tra duplicates (nên = 0)
SELECT url_hash, COUNT(*)
FROM article_metadata
GROUP BY url_hash
HAVING COUNT(*) > 1;
```

## AWS Lambda Version (Production)

Cho AWS deployment, consumer chạy as Lambda function triggered by SQS (không phải Kafka trực tiếp). Xem [Lambda Consumer](5.11-Lambda-Consumer/) cho implementation SQS → Lambda → Aurora.

Khác biệt chính:
- **Trigger**: SQS queue (không phải Kafka)
- **Runtime**: Python 3.11 Lambda (15 min timeout)
- **Database**: Aurora PostgreSQL (không phải local)
- **Secrets**: Lấy từ AWS Secrets Manager
- **Batching**: Xử lý up to 10 messages per invocation

## Xử lý sự cố

| Vấn đề | Giải pháp |
|--------|-----------|
| `Connection refused` to Kafka | Đảm bảo Kafka healthy: `docker compose ps kafka` |
| `Duplicate key value violates unique constraint` | Behavior mong đợi — `ON CONFLICT DO NOTHING` xử lý nó |
| `SSL SYSCALL error: EOF detected` | DB connection dropped; consumer tự reconnect ở poll tiếp theo |
| Consumer lag tăng | Tăng `max_poll_records`, thêm consumer instances (cùng group_id) |
| `Message size too large` | Tăng `message.max.bytes` trong Kafka, hoặc truncate `raw_html` |

## Bước tiếp theo

Sau khi consumer hoạt động cục bộ:
1. **Chạy ETL** để transform raw data → Star Schema: [ETL & Star Schema](5.7-ETL/)
2. **Deploy lên AWS** as Lambda + SQS: [Lambda Consumer](5.11-Lambda-Consumer/)

---

**Tiếp theo:** [ETL & Star Schema](5.7-ETL/)