---
title: "Local Development Setup"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.4 </b> "
---

# Local Development Setup

This section covers setting up the complete local development environment using Docker Compose to simulate the AWS services locally.

## Architecture Overview (Local)

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

## Docker Compose Services

The `docker-compose.yml` defines all services needed for local development:

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

## Starting the Environment

```bash
# Start all services in background
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f postgres
docker compose logs -f kafka
```

Expected output:
```
NAME                 STATUS
newsrag-postgres     Up (healthy)
newsrag-qdrant       Up
newsrag-zookeeper    Up
newsrag-kafka        Up (healthy)
newsrag-kafka-ui     Up
```

## Verifying Services

### PostgreSQL
```bash
# Connect to database
psql -h localhost -U postgres -d newsrag

# Or via Docker
docker exec -it newsrag-postgres psql -U postgres -d newsrag

# Verify tables created from warehouse.sql
\dt
```

Expected tables:
```
         List of relations
 Schema |       Name       | Type  |  Owner
--------+------------------+-------+----------
 public | article_metadata | table | postgres
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
# Check collections
curl http://localhost:6333/collections

# Expected: {"result":{"collections":[]},"status":"ok","time":0.0...}
```

### Kafka
```bash
# List topics
docker exec newsrag-kafka kafka-topics --bootstrap-server localhost:9092 --list

# Create news_raw topic (if not auto-created)
docker exec newsrag-kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --create --topic news_raw \
  --partitions 3 --replication-factor 1
```

### Kafka UI
Open http://localhost:8080 in browser to view topics, messages, consumer groups.

## Python Environment Setup

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows

# Upgrade pip and install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Install package in development mode
pip install -e .
```

### requirements.txt
```txt
# Core
scrapy>=2.11
kafka-python>=2.0
psycopg2-binary>=2.9
qdrant-client>=1.7
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

## Environment Configuration

```bash
# Copy example and edit
cp .env.example .env
```

### .env.example
```env
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

# LLM APIs (Get from respective consoles)
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

# AWS (for future deployment)
AWS_REGION=ap-southeast-2
```

## Running the Local Pipeline

### 1. Crawl News (Scrapy → Kafka)
```bash
# Terminal 1: Start Kafka consumer (writes to PostgreSQL)
python consumer/consumer.py

# Terminal 2: Run crawler
python main.py --mode crawl
```

Expected output:
```
2024-01-15 10:30:45 [INFO] Spider opened: news_sitemap
2024-01-15 10:30:46 [INFO] Crawling VnExpress sitemap: https://vnexpress.net/sitemap_news.xml
2024-01-15 10:30:47 [INFO] Found 150 URLs in sitemap
2024-01-15 10:30:48 [INFO] Parsing article: https://vnexpress.net/kinh-doanh/...
2024-01-15 10:30:49 [INFO] Sent to news_raw[0]@123
...
2024-01-15 10:45:30 [INFO] Closing spider (finished)
```

### 2. Run ETL (PostgreSQL → Star Schema)
```bash
python main.py --mode etl
```

Expected output:
```
2024-01-15 10:46:00 [INFO] ETL processing batch of 50 articles
2024-01-15 10:46:05 [INFO] Upserted dim_source: VnExpress (vnexpress.net)
2024-01-15 10:46:05 [INFO] Upserted dim_time: 2024-01-15 08:30:00+00:00
2024-01-15 10:46:05 [INFO] Upserted dim_content: GDP quý 4 tăng 6.5%...
2024-01-15 10:46:06 [INFO] Inserted fact_articles: 50 articles
2024-01-15 10:46:06 [INFO] Created 327 chunks
2024-01-15 10:46:06 [INFO] ETL batch complete: 50 articles processed
```

### 3. Vectorize (Star Schema → Qdrant)
```bash
python main.py --mode vectorize
```

Expected output:
```
2024-01-15 10:46:30 [INFO] Loading embedding model: BAAI/bge-small-en-v1.5
2024-01-15 10:46:35 [INFO] Model loaded. Embedding dimension: 384
2024-01-15 10:46:35 [INFO] Creating Qdrant collection: news_chunks
2024-01-15 10:46:35 [INFO] Vectorized batch: 256 chunks
2024-01-15 10:46:38 [INFO] Vectorized batch: 71 chunks
2024-01-15 10:46:38 [INFO] Vectorization complete. Total chunks: 327
```

### 4. Run Full Pipeline (Automated)
```bash
python main.py --mode full
```

This runs crawl → etl → vectorize in sequence.

### 5. Test RAG API (Interactive)
```bash
# Start FastAPI server
uvicorn search.api:app --reload --port 8000

# Test in browser
open http://localhost:8000/docs
```

Or via CLI:
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
# Show all commands
make help

# Common commands
make setup              # Create venv, install deps
make up                 # docker compose up -d
make down               # docker compose down -v
make crawl              # python main.py --mode crawl
make etl                # python main.py --mode etl
make vectorize          # python main.py --mode vectorize
make full               # python main.py --mode full
make test-interactive   # Start RAG API for testing
make logs-postgres      # docker compose logs -f postgres
make logs-kafka         # docker compose logs -f kafka
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `PostgreSQL connection refused` | Wait for healthcheck: `docker compose ps` should show `(healthy)` |
| `Kafka not ready` | Wait 30-60s for Kafka to start; check `docker compose logs kafka` |
| `Qdrant collection not found` | Run vectorize step first, or create manually via API |
| `Model download fails` | Check internet; model caches at `~/.cache/huggingface/` |
| `Port already in use` | Stop conflicting services; change ports in docker-compose.yml |
| `CUDA out of memory` | Use CPU: `export CUDA_VISIBLE_DEVICES=""` |

## Development Workflow

```
1. Edit code (crawler/spiders/spider.py, etl/etl_warehouse.py, etc.)
2. Test locally: make crawl / make etl / make vectorize
3. Check logs: docker compose logs -f <service>
4. Verify in DB: psql -h localhost -U postgres -d newsrag
5. Commit changes
6. Build Docker image for AWS deployment (see 5.9-AWS-Deploy)
```

## Next Steps

After local development works:
1. **Build & Push Docker Image** — See [AWS Deployment Prep](5.9-AWS-Deploy/)
2. **Deploy to AWS** — [Fargate Crawler](5.10-Fargate-Crawler/), [Lambda Consumer](5.11-Lambda-Consumer/), [Lambda ETL](5.12-Lambda-ETL/)
3. **Test RAG API** — [RAG API](5.13-RAG-API/)

---

**Next:** [Crawler Development](5.5-Crawler/)