---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---
{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# News RAG Pipeline on AWS — Workshop

## Project Introduction

**News RAG Pipeline on AWS** is a complete end-to-end system that automatically collects news articles from Vietnamese news sites, processes them through a Data Warehouse (Star Schema), generates vector embeddings using Amazon Bedrock, and provides an intelligent Q&A interface using Retrieval-Augmented Generation (RAG) architecture — all running serverless on AWS.

## Architecture Overview

### Local Development (v1 - Docker Compose)
```
Scrapy Crawler → Kafka → PostgreSQL → ETL (Star Schema) → SentenceTransformer → Qdrant → RAG API
```

### AWS Serverless Production (v2 - Current)
```
EventBridge Scheduler (01:00, 02:00 UTC)
       │
       ├──► Fargate Crawler (SitemapSpider) ──► SQS ──► Lambda Consumer ──► Aurora PostgreSQL
       │
       └──► Lambda ETL + Embedding (Bedrock Titan) ──► Aurora pgvector (HNSW Index)
                           │
                           ▼
                    Lambda RAG API ◄── API Gateway ◄── Client
                    (Bedrock embed query → pgvector search → Groq/Gemini LLM)
```

## Key Components

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Scheduler** | EventBridge Scheduler | Daily cron jobs (01:00, 02:00 UTC) |
| **Crawler** | ECS Fargate + Scrapy SitemapSpider | Crawl news sitemaps, push to SQS |
| **Queue** | Amazon SQS Standard | Replace Kafka, ~$0/month |
| **Consumer** | AWS Lambda (SQS trigger) | SHA256 URL dedup, insert raw articles |
| **ETL + Embed** | AWS Lambda (15 min timeout) | Clean HTML → Chunk 500 tokens → Bedrock Titan Embed v2 → pgvector |
| **Warehouse** | Aurora Serverless v2 + pgvector | PostgreSQL + HNSW vector search, 2 ACU |
| **RAG API** | Lambda + API Gateway | Embed query → Vector search → LLM generate |
| **LLM** | Groq (Qwen3-8B) + Gemini 2.0 Flash | External API, no local hosting |
| **Frontend** | Next.js + FastAPI | Dashboard, Search, Chat, Explorer, Monitor |

## Learning Objectives

By the end of this workshop, you will be able to:

1. **Infrastructure as Code**: Define AWS infrastructure using Terraform (VPC, Aurora, ECS, Lambda, EventBridge)
2. **Serverless Containers**: Package and deploy Scrapy crawlers on ECS Fargate with ECR
3. **Data Warehouse Design**: Implement Star Schema in Aurora PostgreSQL with pgvector extension
4. **Workflow Automation**: Schedule daily pipelines using EventBridge Scheduler
5. **RAG System**: Integrate vector database (pgvector) with LLM APIs for intelligent Q&A
6. **Cost Optimization**: Leverage serverless services to minimize operational costs (~$21-26/month)

## Workshop Sections

### Phase 1: Foundation & Infrastructure
1. [Workshop Overview](5.1-Workshop-overview/) - Architecture, objectives, prerequisites, cost estimation
2. [Prerequisites](5.2-Prerequisites/) - AWS CLI, Terraform, Docker, Python, Git setup
3. [Infrastructure as Code (Terraform)](5.3-Infrastructure/) - VPC, Aurora pgvector, ECS, ECR, IAM, EventBridge

### Phase 2: Local Development
4. [Local Development Setup](5.4-Local-Dev/) - Docker Compose (PostgreSQL, Qdrant, Kafka)
5. [Crawler Development](5.5-Crawler/) - Scrapy SitemapSpider for Vietnamese news sites
6. [Data Ingestion](5.6-Ingestion/) - Kafka Consumer → PostgreSQL raw storage
7. [ETL & Star Schema](5.7-ETL/) - Data cleaning, chunking, Star Schema transformation
8. [Vectorization](5.8-Vectorize/) - SentenceTransformer embeddings → Qdrant

### Phase 3: AWS Deployment
9. [AWS Deployment Prep](5.9-AWS-Deploy/) - Docker build, ECR push, ECS Task Definitions
10. [Fargate Crawler](5.10-Fargate-Crawler/) - EventBridge → ECS RunTask → SQS
11. [Lambda Consumer](5.11-Lambda-Consumer/) - SQS Trigger → SHA256 Dedup → Aurora Insert
12. [Lambda ETL + Embedding](5.12-Lambda-ETL/) - Clean → Chunk → Bedrock Titan Embed → pgvector

### Phase 4: RAG Application
13. [RAG API](5.13-RAG-API/) - Lambda + API Gateway → Bedrock Embed Query → pgvector Search → LLM
14. [Frontend Integration](5.14-Frontend/) - Next.js Dashboard, Search, Chat, Explorer, Monitor

### Phase 5: Production Ready
15. [Testing & Monitoring](5.15-Testing/) - RAGAS evaluation, CloudWatch dashboards, alerts
16. [Cost Optimization](5.16-Cost/) - Serverless tuning, Fargate Spot, right-sizing
17. [Clean Up](5.17-Cleanup/) - Terraform destroy, manual cleanup verification

## Project Structure

```
AWS-Projects/
├── config/
│   └── config_site.json          # News site configurations
├── crawler/
│   ├── spiders/spider.py         # Scrapy SitemapSpider
│   ├── pipelines.py              # SQS Pipeline
│   └── settings.py               # Scrapy settings
├── consumer/
│   └── consumer.py               # Lambda Consumer (SQS → Aurora)
├── etl/
│   └── etl_warehouse.py          # ETL + Bedrock Embedding
├── search/
│   ├── engine.py                 # RAG Pipeline
│   ├── retriever.py              # pgvector Search
│   ├── generator.py              # LLM Generation (Groq/Gemini)
│   └── schemas.py                # Pydantic Models
├── database/
│   └── warehouse.sql             # Star Schema + pgvector DDL
├── vectorize/
│   └── vectorize.py              # Legacy Local Vectorization
├── main.py                       # Entry Point (crawl/etl/vectorize/full)
├── main.tf                       # Terraform Infrastructure
├── Dockerfile                    # Multi-stage for ECS Fargate
├── docker-compose.yml            # Local Dev Environment
├── deploy.sh                     # Lambda Deployment Script
├── requirements.txt              # Python Dependencies
└── README.md                     # Project Documentation
```

## Architecture Comparison: v1 vs v2

| Component | v1 (Thesis) | v2 (Workshop) | Reason |
|-----------|-------------|---------------|--------|
| **Crawler** | Lambda + Scrapy (15 min timeout) | **Fargate + SitemapSpider** | No timeout, crawl historical via sitemap |
| **Stream** | Kafka on Docker | **SQS Standard (~$0)** | Simpler, serverless, cost-effective |
| **Embedding** | Local BGE models | **Bedrock Titan Embed v2** | AWS native, serverless, consistent vector space |
| **Vector DB** | Qdrant Cloud | **Aurora + pgvector** | Unified DB, no external dependency |
| **Vectorize** | Separate Fargate Task | **Merged into ETL Lambda** | Fewer services, simpler pipeline |
| **RAG Query Embed** | Local Model (cold start 5-10s) | **Bedrock API** | Consistent embeddings, no cold start |
| **Monthly Cost** | ~$35 | **~$21-26** | ~30% reduction |

> **Key Principle:** ETL and RAG API **must use the same embedding model** (`amazon.titan-embed-text-v2:0`). Different models = different vector spaces = broken search.

## Cost Estimation (Monthly, ap-southeast-2)

| Service | Configuration | Est. Cost |
|---------|---------------|-----------|
| Aurora Serverless v2 | 2 ACU (db.t4g.medium) | ~$15-20 |
| ECS Fargate Crawler | 0.25 vCPU, 0.5 GB, 30 min/day | ~$0.50 |
| Lambda (Consumer, ETL, RAG) | ~1M invocations, 15 min timeout | ~$2-3 |
| SQS Standard | ~30K messages/month | ~$0 |
| API Gateway | ~10K requests/month | ~$0.30 |
| Bedrock Titan Embed | ~500K tokens/month | ~$0.50 |
| CloudWatch Logs | 7-day retention | ~$1-2 |
| **Total** | | **~$21-26/month** |

> Costs vary by region and usage. Check [AWS Pricing Calculator](https://calculator.aws/) for current rates.

## Prerequisites

- **AWS Account** with permissions for: VPC, EC2, RDS, ECS, ECR, Lambda, SQS, EventBridge, API Gateway, Bedrock, CloudWatch, IAM
- **AWS CLI** configured (`aws configure`)
- **Terraform** >= 1.5.0
- **Docker** & **Docker Compose**
- **Python** 3.10+
- **Git**
- **External APIs**: Groq API Key, Google Gemini API Key
- **Bedrock Model Access**: Enable `amazon.titan-embed-text-v2:0` in AWS Console

## Quick Start

```bash
# 1. Clone & Setup
git clone <repo-url> AWS-Projects
cd AWS-Projects

# 2. Local Development
make setup
docker compose up -d
python main.py --mode full

# 3. AWS Deployment
cp .env.example .env
# Edit .env with your values
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars
terraform init
terraform apply

# 4. Build & Push Docker
docker build -t news-crawler .
aws ecr get-login-password | docker login --username AWS --password-stdin <ECR_URL>
docker tag news-crawler:latest <ECR_URL>/news-crawler:latest
docker push <ECR_URL>/news-crawler:latest

# 5. Deploy Lambdas
./deploy.sh

# 6. Test RAG API
curl -X POST <API_URL>/ask -d '{"query": "Tóm tắt tin kinh tế hôm nay"}'
```

---

**Next:** [Workshop Overview](5.1-Workshop-overview/)