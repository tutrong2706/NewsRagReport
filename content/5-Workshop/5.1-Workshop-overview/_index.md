---
title : "Introduction"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

# News RAG Pipeline on AWS — Workshop Overview

This workshop guides you through building a complete **News RAG (Retrieval-Augmented Generation) Pipeline** on AWS using serverless architecture. You will learn to design, deploy, and operate an end-to-end system that automatically crawls news articles, processes them into a Data Warehouse, generates vector embeddings, and enables intelligent Q&A using LLMs.

## Workshop Architecture

![Workshop Architecture](/images/5-Workshop/5.1-Workshop-overview/image.png)
```
EventBridge Scheduler (01:00, 02:00 UTC)
       │
       ├──[01:00]──► Fargate Crawler ──► SQS ──► Lambda Consumer
       │               (Scrapy Sitemap)            (SHA256 + insert Aurora)
       │
       └──[02:00]──► Lambda ETL
                       (clean → chunk → Bedrock Embed → insert Aurora pgvector)

                              ▲
                              │ top-k chunks
                     Lambda RAG API ◄── API Gateway ◄── Client
                     (Bedrock Embed query → pgvector search → Groq/Gemini LLM)
```

## Learning Objectives

By the end of this workshop, you will be able to:

1. **Infrastructure as Code**: Define AWS infrastructure using Terraform (VPC, Aurora, ECS, Lambda, EventBridge)
2. **Serverless Containers**: Package and deploy Scrapy crawlers on ECS Fargate with ECR
3. **Data Warehouse Design**: Implement Star Schema in Aurora PostgreSQL with pgvector extension
4. **Workflow Automation**: Schedule daily pipelines using EventBridge Scheduler
5. **RAG System**: Build retrieval-augmented generation with Bedrock embeddings and external LLMs
6. **Cost Optimization**: Leverage serverless services to minimize operational costs (~$21-26/month)

## Workshop Modules

| Module | Title | Description | Duration |
|--------|-------|-------------|----------|
| 5.1 | **Workshop Overview** | Architecture, objectives, prerequisites, cost estimation | 30 min |
| 5.2 | **Prerequisites** | AWS CLI, Terraform, Docker, Python setup | 30 min |
| 5.3 | **Infrastructure (Terraform)** | VPC, Aurora pgvector, ECS Cluster, ECR, IAM, EventBridge | 60 min |
| 5.4 | **Crawler (ECS Fargate)** | Dockerfile, Scrapy SitemapSpider, ECR push, Task Definition | 45 min |
| 5.5 | **Queue & Consumer (SQS + Lambda)** | SQS queue, Lambda Consumer, SHA256 dedup, Aurora insert | 45 min |
| 5.6 | **ETL + Embedding (Lambda + Bedrock)** | HTML clean, chunking, Titan Embed v2, pgvector HNSW insert | 60 min |
| 5.7 | **RAG API (Lambda + API Gateway)** | Query embed, pgvector search, LLM generation, response | 45 min |
| 5.8 | **Frontend Integration** | Next.js Dashboard, Search, Chat, Explorer, Pipeline Monitor | 60 min |
| 5.9 | **Testing & Monitoring** | CloudWatch Logs, query testing, RAG evaluation | 30 min |
| 5.10 | **Clean Up** | Terraform destroy, resource cleanup | 15 min |

## Prerequisites

- **AWS Account** with permissions for: VPC, EC2, RDS, ECS, ECR, Lambda, SQS, EventBridge, API Gateway, Bedrock, CloudWatch, IAM
- **AWS CLI** configured (`aws configure`) with appropriate credentials
- **Terraform** >= 1.5.0 installed
- **Docker** & **Docker Compose** installed
- **Python** 3.10+ with pip
- **Git** for version control
- **Code Editor** (VS Code recommended)

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

> **Note:** Costs vary by region and usage. Check [AWS Pricing Calculator](https://calculator.aws/) for current rates.

## Project Structure

```
AWS-Projects/
├── config/
│   └── config_site.json          # News site configurations
├── crawler/
│   ├── spiders/spider.py         # Scrapy SitemapSpider
│   ├── pipelines.py              # SQS pipeline
│   └── settings.py               # Scrapy settings
├── consumer/
│   └── consumer.py               # Lambda Consumer (SQS → Aurora)
├── etl/
│   └── etl_warehouse.py          # ETL + Bedrock Embedding
├── search/
│   ├── engine.py                 # RAG Pipeline
│   ├── retriever.py              # pgvector search
│   ├── generator.py              # LLM generation (Groq/Gemini)
│   └── schemas.py                # Pydantic models
├── database/
│   └── warehouse.sql             # Star Schema + pgvector
├── vectorize/
│   └── vectorize.py              # Legacy vectorization
├── main.py                       # Entry point (crawl/etl/vectorize/full)
├── main.tf                       # Terraform infrastructure
├── Dockerfile                    # Multi-stage for ECS Fargate
├── docker-compose.yml            # Local dev (PostgreSQL, Qdrant, Kafka)
├── deploy.sh                     # Lambda deployment script
├── requirements.txt              # Python dependencies
└── README.md                     # Project documentation
```

## Architecture Comparison: v1 vs v2

| Component | v1 (Thesis) | v2 (This Workshop) | Reason |
|-----------|-------------|-------------------|--------|
| **Crawler** | Lambda + Scrapy (15 min timeout) | **Fargate + SitemapSpider** | No timeout, crawl historical via sitemap |
| **Stream** | Kafka on Docker | **SQS Standard (~$0)** | Simpler, serverless, cost-effective |
| **Embedding** | Local BGE models | **Bedrock Titan Embed v2** | AWS native, serverless, consistent vector space |
| **Vector DB** | Qdrant Cloud | **Aurora + pgvector** | Unified DB, no external dependency |
| **Vectorize** | Separate Fargate task | **Merged into ETL Lambda** | Fewer services, simpler pipeline |
| **RAG Query Embed** | Local model (cold start 5-10s) | **Bedrock API** | Consistent embeddings, no cold start |
| **Monthly Cost** | ~$35 | **~$21-26** | ~30% reduction |

> **Key Principle:** ETL and RAG API **must use the same embedding model** (`amazon.titan-embed-text-v2:0`). Different models = different vector spaces = broken search.

---

**Next:** [Prerequisites](5.2-Prerequisites/)