---
title: "Phát triển Crawler"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.5 </b> "
---

# Phát triển Crawler

Phần này bao gồm việc xây dựng crawler Scrapy để thu thập bài báo từ các trang tin tức Việt Nam sử dụng SitemapSpider.

## Tại sao dùng SitemapSpider?

Khác với `CrawlSpider` truyền thống dùng `DEPTH_LIMIT`, `SitemapSpider` đọc trực tiếp `sitemap_news.xml`, đảm bảo:
- **Không bỏ sót bài** — phát hiện tất cả URL trong sitemap, không chỉ các trang link
- **Truy cập lịch sử** — có thể crawl bài cũ qua sitemap archives
- **Phát hiện nhanh hơn** — không cần follow pagination links
- **Tuân thủ robots.txt** — built-in `ROBOTSTXT_OBEY = True`

## Cấu trúc dự án

```
crawler/
├── __init__.py
├── settings.py           # Cài đặt Scrapy
├── pipelines.py          # Item pipelines (Kafka, PostgreSQL)
├── items.py              # Data models
└── spiders/
    ├── __init__.py
    └── spider.py         # SitemapSpider implementation
```

## Cấu hình (config/config_site.json)

```json
[
  {
    "name": "VnExpress",
    "url": "https://vnexpress.net/sitemap_news.xml",
    "domain": "vnexpress.net",
    "category_xpath": "//meta[@property='article:section']/@content",
    "title_xpath": "//h1[@class='title-detail']//text()",
    "content_xpath": "//article[@class='fck_detail']//p//text()",
    "author_xpath": "//meta[@name='author']/@content",
    "published_xpath": "//meta[@property='article:published_time']/@content"
  },
  {
    "name": "ThanhNien",
    "url": "https://thanhnien.vn/sitemap.xml",
    "domain": "thanhnien.vn",
    "category_xpath": "//meta[@property='article:section']/@content",
    "title_xpath": "//h1[@class='detail-title']//text()",
    "content_xpath": "//div[@class='detail-content']//p//text()",
    "author_xpath": "//meta[@name='author']/@content",
    "published_xpath": "//meta[@property='article:published_time']/@content"
  },
  {
    "name": "VietnamNet",
    "url": "https://vietnamnet.vn/sitemap_news.xml",
    "domain": "vietnamnet.vn",
    "category_xpath": "//meta[@property='article:section']/@content",
    "title_xpath": "//h1[@class='title-detail']//text()",
    "content_xpath": "//div[@class='main-content']//p//text()",
    "author_xpath": "//meta[@name='author']/@content",
    "published_xpath": "//meta[@property='article:published_time']/@content"
  }
]
```

Mỗi trang định nghĩa XPath selectors để trích xuất dữ liệu có cấu trúc.

## Cài đặt Scrapy (crawler/settings.py)

```python
BOT_NAME = "newsrag_crawler"

SPIDER_MODULES = ["crawler.spiders"]
NEWSPIDER_MODULE = "crawler.spiders"

# Tuân thủ robots.txt
ROBOTSTXT_OBEY = True

# Concurrency & Delay (lịch sự)
CONCURRENT_REQUESTS = 8
CONCURRENT_REQUESTS_PER_DOMAIN = 4
DOWNLOAD_DELAY = 1
RANDOMIZE_DOWNLOAD_DELAY = 0.5

# Timeout
DOWNLOAD_TIMEOUT = 30

# User Agent
USER_AGENT = "NewsRAG Bot (+https://github.com/your-repo/newsrag)"

# AutoThrottle (delay thích ứng)
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 1
AUTOTHROTTLE_MAX_DELAY = 10
AUTOTHROTTLE_TARGET_CONCURRENCY = 4.0

# Pipelines
ITEM_PIPELINES = {
    "crawler.pipelines.KafkaPipeline": 300,
}

# Kafka Configuration
KAFKA_BOOTSTRAP_SERVERS = "localhost:9092"
KAFKA_TOPIC_NEWS = "news_raw"

# Logging
LOG_LEVEL = "INFO"
LOG_FORMAT = "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
```

## Items (crawler/items.py)

```python
import scrapy


class NewsArticleItem(scrapy.Item):
    """Raw article data từ crawler"""
    url = scrapy.Field()
    url_hash = scrapy.Field()        # SHA256 cho deduplication
    source_name = scrapy.Field()     # VnExpress, ThanhNien, etc.
    source_domain = scrapy.Field()   # vnexpress.net
    title = scrapy.Field()
    content = scrapy.Field()
    category = scrapy.Field()
    author = scrapy.Field()
    published_at = scrapy.Field()    # ISO datetime string
    crawled_at = scrapy.Field()      # UTC now
    raw_html = scrapy.Field()        # Full HTML cho debugging
```

## SitemapSpider Implementation (crawler/spiders/spider.py)

```python
import hashlib
import json
from datetime import datetime, timezone
from typing import Optional

import scrapy
from scrapy.spiders import SitemapSpider

from crawler.items import NewsArticleItem
from crawler.settings import KAFKA_BOOTSTRAP_SERVERS, KAFKA_TOPIC_NEWS


class NewsSitemapSpider(SitemapSpider):
    """
    Crawl các trang tin tức Việt Nam qua sitemap_news.xml
    Đẩy bài viết trích xuất được vào Kafka để xử lý tiếp.
    """
    name = "news_sitemap"

    # Load site configs từ JSON
    with open("config/config_site.json", "r", encoding="utf-8") as f:
        SITE_CONFIGS = json.load(f)

    # Xây dựng sitemap_urls và rules động
    sitemap_urls = [site["url"] for site in SITE_CONFIGS]
    sitemap_rules = [("/", "parse_article")]  # Match tất cả URL trong sitemap

    custom_settings = {
        "CLOSESPIDER_TIMEOUT": 1800,  # 30 phút max mỗi spider run
        "CLOSESPIDER_ITEMCOUNT": 1000,  # Max bài viết mỗi run
    }

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.site_config_map = {site["domain"]: site for site in self.SITE_CONFIGS}
        self.kafka_producer = None
        self._init_kafka()

    def _init_kafka(self):
        """Khởi tạo Kafka producer"""
        try:
            from kafka import KafkaProducer
            self.kafka_producer = KafkaProducer(
                bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
                value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
                key_serializer=lambda k: k.encode("utf-8") if k else None,
                acks="all",
                retries=3,
                max_in_flight_requests_per_connection=1,
            )
        except Exception as e:
            self.logger.warning(f"Kafka producer init failed: {e}")
            self.kafka_producer = None

    def parse_article(self, response):
        """Parse trang bài viết dùng site-specific XPath selectors"""
        domain = self._extract_domain(response.url)
        config = self.site_config_map.get(domain)

        if not config:
            self.logger.warning(f"No config for domain: {domain}")
            return

        # Trích xuất fields dùng site-specific XPaths
        title = self._extract_text(response, config.get("title_xpath"))
        content = self._extract_text(response, config.get("content_xpath"))
        category = self._extract_text(response, config.get("category_xpath"))
        author = self._extract_text(response, config.get("author_xpath"))
        published_str = self._extract_text(response, config.get("published_xpath"))

        # Validate required fields
        if not title or not content:
            self.logger.debug(f"Skipping {response.url}: missing title or content")
            return

        # Parse published date
        published_at = self._parse_date(published_str)

        # Tạo item
        item = NewsArticleItem()
        item["url"] = response.url
        item["url_hash"] = hashlib.sha256(response.url.encode()).hexdigest()
        item["source_name"] = config["name"]
        item["source_domain"] = domain
        item["title"] = title.strip()
        item["content"] = " ".join(content.split())  # Normalize whitespace
        item["category"] = category.strip() if category else "Unknown"
        item["author"] = author.strip() if author else "Unknown"
        item["published_at"] = published_at.isoformat() if published_at else datetime.now(timezone.utc).isoformat()
        item["crawled_at"] = datetime.now(timezone.utc).isoformat()
        item["raw_html"] = response.text[:50000]  # Giới hạn size

        # Gửi tới Kafka (non-blocking)
        if self.kafka_producer:
            try:
                future = self.kafka_producer.send(
                    KAFKA_TOPIC_NEWS,
                    key=item["url_hash"],
                    value=dict(item)
                )
                # Optional: wait for confirmation
                # future.get(timeout=5)
            except Exception as e:
                self.logger.error(f"Kafka send failed: {e}")

        yield item

    def _extract_domain(self, url: str) -> str:
        """Trích xuất domain từ URL"""
        from urllib.parse import urlparse
        return urlparse(url).netloc.replace("www.", "")

    def _extract_text(self, response, xpath: Optional[str]) -> Optional[str]:
        """Trích xuất và join text từ XPath"""
        if not xpath:
            return None
        texts = response.xpath(xpath).getall()
        return " ".join(t.strip() for t in texts if t.strip()) if texts else None

    def _parse_date(self, date_str: Optional[str]) -> Optional[datetime]:
        """Parse các định dạng date khác nhau"""
        if not date_str:
            return None
        formats = [
            "%Y-%m-%dT%H:%M:%S%z",      # ISO with timezone
            "%Y-%m-%dT%H:%M:%S",        # ISO without timezone
            "%Y-%m-%d %H:%M:%S",        # Standard
            "%d/%m/%Y %H:%M",           # VN format
            "%Y-%m-%d",                 # Date only
        ]
        for fmt in formats:
            try:
                return datetime.strptime(date_str.strip(), fmt)
            except ValueError:
                continue
        self.logger.debug(f"Could not parse date: {date_str}")
        return None

    def closed(self, reason):
        """Cleanup khi spider đóng"""
        if self.kafka_producer:
            self.kafka_producer.flush()
            self.kafka_producer.close()
        self.logger.info(f"Spider closed: {reason}")
```

## Kafka Pipeline (crawler/pipelines.py)

```python
import json
import logging
from kafka import KafkaProducer
from kafka.errors import KafkaError

from crawler.settings import KAFKA_BOOTSTRAP_SERVERS, KAFKA_TOPIC_NEWS


class KafkaPipeline:
    """Scrapy pipeline gửi items đến Kafka"""

    def __init__(self):
        self.producer = None
        self.logger = logging.getLogger(__name__)

    def open_spider(self, spider):
        try:
            self.producer = KafkaProducer(
                bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
                value_serializer=lambda v: json.dumps(v, ensure_ascii=False).encode("utf-8"),
                key_serializer=lambda k: k.encode("utf-8") if k else None,
                acks="all",
                retries=3,
                max_in_flight_requests_per_connection=1,
            )
            self.logger.info("Kafka producer initialized")
        except Exception as e:
            self.logger.error(f"Failed to initialize Kafka producer: {e}")

    def close_spider(self, spider):
        if self.producer:
            self.producer.flush()
            self.producer.close()
            self.logger.info("Kafka producer closed")

    def process_item(self, item, spider):
        if not self.producer:
            return item

        try:
            article_dict = dict(item)
            url_hash = article_dict.pop("url_hash", None)

            future = self.producer.send(
                KAFKA_TOPIC_NEWS,
                key=url_hash,
                value=article_dict
            )
            # Non-blocking - errors handled in callback
            future.add_callback(self._on_success)
            future.add_errback(self._on_error)
        except Exception as e:
            self.logger.error(f"Kafka send error: {e}")

        return item

    def _on_success(self, metadata):
        self.logger.debug(f"Sent to {metadata.topic}[{metadata.partition}]@{metadata.offset}")

    def _on_error(self, exc):
        self.logger.error(f"Kafka send failed: {exc}")
```

## Chạy Crawler

### Local Development (với Docker Compose)

```bash
# 1. Khởi động services
docker compose up -d

# 2. Tạo Kafka topic
docker exec newsrag-kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --create --topic news_raw --partitions 3 --replication-factor 1

# 3. Chạy crawler
cd crawler
scrapy crawl news_sitemap -o output.json

# Hoặc qua main.py
python ../main.py --mode crawl
```

### Output mong đợi

```
2024-01-15 10:30:45 [INFO] Spider opened: news_sitemap
2024-01-15 10:30:46 [INFO] Crawling VnExpress sitemap: https://vnexpress.net/sitemap_news.xml
2024-01-15 10:30:47 [INFO] Found 150 URLs in sitemap
2024-01-15 10:30:48 [INFO] Parsing article: https://vnexpress.net/kinh-doanh/...
2024-01-15 10:30:49 [INFO] Sent to news_raw[0]@123
...
2024-01-15 10:45:30 [INFO] Closing spider (finished)
2024-01-15 10:45:30 [INFO] Spider closed: finished
```

### Xác minh Kafka Messages

```bash
# Consume messages
docker exec newsrag-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic news_raw \
  --from-beginning \
  --max-messages 5
```

Sample message:
```json
{
  "url": "https://vnexpress.net/kinh-doanh/...",
  "url_hash": "a1b2c3d4...",
  "source_name": "VnExpress",
  "source_domain": "vnexpress.net",
  "title": "GDP quý 4 tăng 6.5%...",
  "content": "Theo Tổng cục Thống kê...",
  "category": "Kinh doanh",
  "author": "Phạm Du",
  "published_at": "2024-01-15T08:30:00+07:00",
  "crawled_at": "2024-01-15T03:30:45+00:00"
}
```

## Dockerfile cho Fargate (Multi-stage)

```dockerfile
# Dockerfile
FROM python:3.11-slim as builder

WORKDIR /app
RUN pip install --no-cache-dir --upgrade pip

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && \
    rm -rf /var/lib/apt/lists/*

# Copy installed packages
COPY --from=builder /install /usr/local

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Default command (ghi đè bởi ECS task definition)
CMD ["python", "main.py", "--mode", "crawl"]
```

## Build & Test cục bộ

```bash
# Build image
docker build -t news-crawler:local .

# Chạy container (cần Kafka, PostgreSQL chạy)
docker run --rm \
  --network host \
  -e KAFKA_BOOTSTRAP_SERVERS=localhost:9092 \
  -e DB_HOST=localhost \
  news-crawler:local python main.py --mode crawl
```

## Thêm trang tin tức mới

1. **Phân tích trang** — Tìm `sitemap_news.xml` (thường ở `/sitemap_news.xml` hoặc `/sitemap.xml`)
2. **Kiểm tra trang bài viết** — Dùng browser DevTools tìm XPath selectors cho title, content, author, date, category
3. **Thêm vào config_site.json** — Copy entry hiện có và sửa selectors
4. **Test cục bộ** — Chạy spider và xác minh trích xuất
5. **Deploy** — Rebuild Docker image, push lên ECR

## Vấn đề thường gặp & Khắc phục

| Vấn đề | Giải pháp |
|--------|-----------|
| `403 Forbidden` | Thêm headers phù hợp, tuân thủ `ROBOTSTXT_OBEY`, giảm `CONCURRENT_REQUESTS` |
| `XPath returns empty` | Kiểm tra site có dùng JavaScript rendering (cần Splash/Playwright) |
| `Kafka connection failed` | Đảm bảo Kafka healthy, check `bootstrap.servers` |
| `Memory error` | Giảm `CLOSESPIDER_ITEMCOUNT`, tăng container memory |
| `Timeout` | Tăng `DOWNLOAD_TIMEOUT`, `CLOSESPIDER_TIMEOUT` |

## Bước tiếp theo

Sau khi crawler hoạt động cục bộ:
1. **Build & Push lên ECR** — Xem [AWS Deployment Prep](5.9-AWS-Deploy/)
2. **Đăng ký Task Definition** — Xem [Fargate Crawler](5.10-Fargate-Crawler/)
3. **Cấu hình SQS** — Thay Kafka bằng SQS cho AWS deployment (xem [Lambda Consumer](5.11-Lambda-Consumer/))

---

**Tiếp theo:** [Nhập dữ liệu (Kafka Consumer → PostgreSQL)](5.6-Ingestion/)