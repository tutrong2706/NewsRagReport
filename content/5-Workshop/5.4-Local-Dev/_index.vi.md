---
title: "Thiết lập phát triển cục bộ"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 5.4 </b> "
---

# Thiết lập phát triển cục bộ

Phần này bao gồm việc thiết lập môi trường phát triển cục bộ hoàn chỉnh sử dụng Docker Compose để mô phỏng các dịch vụ AWS cục bộ.

## Tổng quan kiến trúc (Local)

```
┌─────────────┐     ┌─────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│   Scrapy    │────►│  Kafka  │────►│  PostgreSQL  │────►│    ETL      │────►│  Qdrant  │
│  Crawler    │     │         │     │  (Raw Data)  │     │(Star Schema)│     │ (Vectors)│
└─────────────┘     └─────────┘     └──────────────┘     └─────────────┘     └──────────┘
                           │                                     │                    │
                           │                                     │                    │
                           ▼                                     ▼                    ▼
                    ┌─────────────┐                       ┌─────────────┐     ┌─────────────┐
                    │  Zookeeper  │                       │  Sentence   │     │  RAG API    │
                    │             │                       │  Transformer│     │  (FastAPI)  │
                    └─────────────┘                       └─────────────┘     └─────────────┘
```

## Các dịch vụ Docker Compose

File `docker-compose.yml` định nghĩa tất cả các dịch vụ cần thiết cho phát triển cục bộ:

```yaml
version: '3.8'

services:
  # PostgreSQL with pgvector extension
  postgres:
    image: pgvector/pgvector:pg15
    container_name: newsrag-postgres
    environment:
      POSTGRES_DB: newsrag
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/warehouse.sql:/docker-entrypoint-initdb.d/warehouse.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:latest
    container_name: newsrag-qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage

  # Kafka + Zookeeper
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: newsrag-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: newsrag-kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  # Optional: Kafka UI for monitoring
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: newsrag-kafka-ui
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181

volumes:
  postgres_data:
  qdrant_data:
```

## Khởi động môi trường

```bash
# Khởi động tất cả dịch vụ nền
docker compose up -d

# Kiểm tra trạng thái
docker compose ps

# Xem logs
docker compose logs -f postgres
docker compose logs -f kafka
```

Kết quả mong đợi:
```
NAME                 STATUS
newsrag-postgres     Up (healthy)
newsrag-qdrant       Up
newsrag-zookeeper    Up
newsrag-kafka        Up (healthy)
newsrag-kafka-ui     Up
```

## Xác minh các dịch vụ

### PostgreSQL
```bash
# Kết nối database
psql -h localhost -U postgres -d newsrag

# Hoặc qua Docker
docker exec -it newsrag-postgres psql -U postgres -d newsrag

# Xác minh tables được tạo từ warehouse.sql
\dt
```

Tables mong đợi:
```
         List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | dim_author       | table | postgres
 public | dim_content      | table | postgres
 public | dim_source       | table | postgres
 public | dim_time         | table | postgres
 public | fact_article_authors | table | postgres
 public | fact_articles    | table | postgres
 public | fact_chunks      | table | postgres
```

### Qdrant
```bash
# Kiểm tra collections
curl http://localhost:6333/collections

# Kết quả mong đợi: {"result":{"collections":[]},"status":"ok","time":0.0...}
```

### Kafka
```bash
# Liệt kê topics
docker exec newsrag-kafka kafka-topics --bootstrap-server localhost:9092 --list

# Tạo news_raw topic (nếu chưa auto-created)
docker exec newsrag-kafka kafka-topics --bootstrap-server localhost:9092 --create --topic news_raw --partitions 3 --replication-factor 1
```

### Kafka UI
Mở http://localhost:8080 trong trình duyệt để xem topics, messages, consumer groups.

## Thiết lập môi trường Python

```bash
# Tạo virtual environment
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows

# Nâng cấp pip và cài đặt dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Cài đặt package ở chế độ development
pip install -e .
```

### requirements.txt
```txt
# Core
scrapy>=2.11
kafka-python>=2.0
psycopg2-binary>=2.9
qdrant-client>=1.7

# ETL & NLP
beautifulsoup4>=4.12
newspaper3k>=0.2.8
sentence-transformers>=2.2
langchain>=0.1
langchain-community>=0.0

# RAG & LLM
groq>=0.4
google-generativeai>=0.3
pydantic>=2.5
pydantic-settings>=2.1

# Utilities
python-dotenv>=1.0
schedule>=1.2
tqdm>=4.66

# API
fastapi>=0.104
uvicorn>=0.24

# Testing
pytest>=7.4
pytest-asyncio>=0.21
```

## Cấu hình biến môi trường

```bash
# Sao chép file mẫu
cp .env.example .env

# Chỉnh sửa .env với giá trị của bạn
cat > .env << 'EOF'
# Database (Local PostgreSQL)
DB_NAME=newsrag
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=localhost
DB_PORT=5432

# Qdrant (Local)
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_COLLECTION_NAME=news_chunks
QDRANT_API_KEY=

# Kafka (Local)
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_TOPIC_NEWS=news_raw

# Embedding Model (Local SentenceTransformer)
EMBEDDING_MODEL=BAAI/bge-small-en-v1.5
EMBEDDING_SIZE=384

# LLM APIs (Lấy từ respective consoles)
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxx
GEMINI_API_KEY=AIza_xxxxxxxxxxxxxxxxxxxx

# Model Config
NUM_MODEL_SUPPORT=3
MODEL_1_NAME=qwen3-8b-instant
MODEL_1_MODEL_ID=qwen/qwen3-8b-instant
MODEL_1_PROVIDER=groq
MODEL_1_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxx
MODEL_2_NAME=llama-3.1-8b-instant
MODEL_2_MODEL_ID=meta-llama/llama-3.1-8b-instant
MODEL_2_PROVIDER=groq
MODEL_2_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxx
MODEL_3_NAME=gemini-2.0-flash
MODEL_3_MODEL_ID=gemini-2.0-flash
MODEL_3_PROVIDER=google
MODEL_3_API_KEY=AIza_xxxxxxxxxxxxxxxxxxxx

# AWS
AWS_REGION=ap-southeast-2
EOF
```

## Chạy Pipeline cục bộ

### 1. Crawl tin tức (Scrapy → Kafka)
```bash
# Terminal 1: Khởi động Kafka Consumer (ghi vào PostgreSQL)
python consumer/consumer.py

# Terminal 2: Chạy Crawler
python main.py --mode crawl
```

Crawler:
- Đọc `config/config_site.json` để lấy danh sách URL báo
- Chạy Scrapy SitemapSpider cho mỗi trang
- Đẩy dữ liệu bài viết vào Kafka topic `news_raw`

### 2. Chạy ETL (PostgreSQL → Star Schema)
```bash
# Sau khi crawler xong và consumer đã xử lý messages
python main.py --mode etl
```

ETL process:
- Đọc bài thô từ bảng `article_metadata`
- Làm sạch HTML, trích xuất text
- Chunk text thành các đoạn ~500 token
- Điền Star Schema tables (`dim_*`, `fact_*`)
- Cập nhật `embedded = true` trên bài đã xử lý

### 3. Vectorize (Star Schema → Qdrant)
```bash
python main.py --mode vectorize
```

Vectorizer:
- Đọc chunks từ bảng `fact_chunks`
- Tạo embeddings bằng SentenceTransformer (model local)
- Upsert vectors vào Qdrant collection `news_chunks`

### 4. Chạy Full Pipeline (Tự động)
```bash
# Chạy crawl → ETL → vectorize lần lượt
python main.py --mode full
```

### 5. Test RAG API (Interactive)
```bash
# Khởi động FastAPI server
uvicorn search.api:app --reload --port 8000

# Hoặc dùng Makefile
make test-interactive
```

Test trên trình duyệt: http://localhost:8000/docs (Swagger UI)

Hoặc CLI:
```bash
python -c "
from search.engine import Pipeline
p = Pipeline()
result = p.ask('Tóm tắt tin tức về kinh tế Việt Nam hôm nay')
print(result.summary)
print(f'Sources: {result.total}, Time: {result.duration_ms}ms')
"
```

## Makefile Commands

```bash
# Xem tất cả lệnh
make help

# Các lệnh thường dùng
make setup              # Tạo venv, cài deps
make up                 # docker compose up -d
make down               # docker compose down -v
make crawl              # python main.py --mode crawl
make etl                # python main.py --mode etl
make vectorize          # python main.py --mode vectorize
make full               # python main.py --mode full
make test-interactive   # Khởi động RAG API để test
make logs-postgres      # docker compose logs -f postgres
make logs-kafka         # docker compose logs -f kafka
```

## Xử lý sự cố

| Vấn đề | Giải pháp |
|--------|-----------|
| `PostgreSQL connection refused` | Kiểm tra `docker compose ps postgres`, chờ healthy. Xem `docker compose logs postgres` |
| `Kafka not ready` | Đợi Kafka healthy (30-60s). Kiểm tra `docker compose logs -f kafka` |
| `Qdrant collection issues` | Xóa và tạo lại collection: `curl -X DELETE http://localhost:6333/collections/news_chunks` |
| `SentenceTransformer model download` | Lần đầu tải model (~100MB). Lần sau dùng cache tại `~/.cache/huggingface/` |
| `Port conflicts` | Kiểm tra port 5432, 6333, 9092, 8080. Dừng dịch vụ xung đột hoặc đổi port trong docker-compose.yml |

## Quy trình phát triển

```
1. Sửa code (crawler/spiders/spider.py, etl/etl_warehouse.py, etc.)
2. Test cục bộ: make crawl / make etl / make vectorize
3. Kiểm tra logs: docker compose logs -f <service>
4. Xác minh DB: psql -h localhost -U postgres -d newsrag -c "SELECT * FROM fact_chunks LIMIT 5;"
5. Test RAG: make test-interactive
6. Commit changes
7. Build Docker image cho AWS deployment (xem 5.9-AWS-Deploy)
```

## Bước tiếp theo

Sau khi phát triển cục bộ hoạt động:
1. **Build Docker image** cho AWS: [AWS Deployment Prep](5.9-AWS-Deploy/)
2. **Deploy lên AWS** — [Fargate Crawler](5.10-Fargate-Crawler/), [Lambda Consumer](5.11-Lambda-Consumer/), [Lambda ETL](5.12-Lambda-ETL/)
3. **Test RAG API** — [RAG API](5.13-RAG-API/)

---

**Tiếp theo:** [Phát triển Crawler](5.5-Crawler/)