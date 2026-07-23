---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# News RAG — Intelligent News Aggregation and Analysis System
## AWS Serverless Solution for News Aggregation with RAG Architecture

### 1. Executive Summary

**News RAG** is an intelligent news aggregation and analysis system leveraging RAG (Retrieval-Augmented Generation) architecture to automatically collect news from Vietnamese newspapers (VnExpress, Thanh Niên, VietnamNet), process and store data using Star Schema, create embedding vectors, and enable users to ask questions in natural language — the system retrieves relevant news chunks and synthesizes answers through Large Language Models (LLMs) such as Groq Qwen3 and Gemini Flash.

The project was developed by a team of 4 members in the **First Cloud AI Journey** internship program at **AWS Vietnam**, fully deployed on AWS with serverless architecture, optimized for cost (~$21-26/month).

### 2. Problem Statement

*Current Problem*
Vietnamese users face difficulties in tracking and analyzing information from multiple news sources. Manually reading hundreds of articles daily is extremely time-consuming. No existing system enables intelligent Q&A based on Vietnamese news data using RAG technology.

*Solution*
The NewsRAG system automatically:
1. **Crawls** news from 3 newspapers via Scrapy SitemapSpider on ECS Fargate
2. **Streams** data through SQS → Lambda Consumer → PostgreSQL
3. **ETL** standardizes data into Star Schema, chunks text at 800 chars
4. **Vectorizes** embeddings via Bedrock Titan Embed v2 (1024 dimensions)
5. **RAG** enables intelligent Q&A: embed query → vector search → LLM generate

*Benefits*
- Saves users time in monitoring news
- Provides answers with source citations, increasing reliability
- Low operational cost (~$21-26/month) thanks to serverless architecture
- Scalable to additional news sources and languages

### 3. Solution Architecture

```text
EventBridge Scheduler (01:00, 02:00, 03:00 UTC)
       │
       ├──[01:00]──► Fargate Crawler ──► SQS ──► Lambda Consumer
       │               (Scrapy Sitemap)            (SHA256 + insert Aurora)
       │
       ├──[02:00]──► Fargate ETL
       │               (clean → chunk → Star Schema)
       │
       └──[03:00]──► Fargate Vectorize
                       (Bedrock Embed → Qdrant)

                              ▲
                              │ top-k chunks
                     Lambda RAG API ◄── API Gateway ◄── Client
                     (Embed query → vector search → Groq/Gemini LLM)
```

*AWS Services Used*
- **ECS Fargate**: Run container Crawler, ETL, Vectorize (no server management)
- **Amazon SQS**: Message queue replacing Kafka (~$0/month)
- **Amazon ECR**: Docker image registry
- **RDS Aurora PostgreSQL**: Article storage + Star Schema warehouse
- **Amazon Bedrock**: Titan Embed Text v2 (1024 dimensions) — serverless embedding
- **EventBridge Scheduler**: Automated pipeline scheduling
- **CloudWatch**: Log monitoring
- **VPC + Security Groups**: Secure networking

*External Services*
- **Qdrant Cloud**: Vector database for similarity search
- **Groq API** (Qwen3-8B): Primary LLM for RAG
- **Google Gemini 2.0 Flash**: Fallback LLM

### 4. Technical Implementation

*Team Task Distribution*

| Member     | Module              | AWS Services                                 | Main Tasks                                                |
| ---------- | ------------------- | -------------------------------------------- | --------------------------------------------------------- |
| **A (Me)** | Crawl + Queue       | ECS Fargate, SQS, ECR, Docker                | Dockerfile, SQS + DLQ, ECR deploy, Sitemap Spider         |
| **B**      | Consumer + Database | Lambda, Aurora, pgvector, Secrets Manager    | Lambda Consumer, Aurora cluster, Star Schema SQL           |
| **C**      | ETL + Embedding     | Lambda, Bedrock, S3                          | Clean HTML, chunk text, Bedrock Embed, insert vector       |
| **D**      | RAG API + Frontend  | Lambda, API Gateway, Groq/Gemini, Next.js    | RAG search, API Gateway, Frontend Dashboard                |

*Technologies Used*
- **Backend**: Python 3.10, Scrapy, psycopg2, boto3, sentence-transformers
- **Frontend**: Next.js, React, Tailwind CSS, FastAPI
- **Infrastructure**: Terraform, Docker, Docker Compose
- **Database**: PostgreSQL 15 + pgvector extension
- **LLM**: LangChain + Groq + Google GenAI

### 5. Roadmap & Timeline (8 Weeks)

| Week    | Main Tasks                                                  |
| ------- | ----------------------------------------------------------- |
| **1**   | AWS orientation, understand NewsRAG architecture, setup env |
| **2**   | Research Scrapy, Docker, analyze sitemap XML                |
| **3**   | Develop NewsRAGSpider, KafkaPipeline                        |
| **4**   | Dockerize Crawler, push ECR, configure ECS Fargate          |
| **5**   | SQS + DLQ, integrate Crawler → Consumer → PostgreSQL        |
| **6**   | Terraform infrastructure, EventBridge Schedule              |
| **7**   | Integration testing, debug edge cases, Ragas evaluation     |
| **8**   | Finalize code, write documentation, internship report       |

### 6. Budget Estimation

| Group             | Component                                              | Cost/Month        |
| ----------------- | ------------------------------------------------------ | ----------------- |
| **Crawl + Queue** | Fargate (0.25 vCPU, 512 MB, ~30 min/day) + SQS + Consumer | **~$3–5**     |
| **ETL + Embed**   | Fargate ETL + Bedrock Titan Embed                      | **~$1–3**         |
| **Database**      | Aurora Serverless v2 (2 ACU) + pgvector                | **~$14**          |
| **RAG API**       | Lambda RAG + API Gateway                               | **~$3–4**         |
| **Total**         |                                                        | **~$21–26/month** |

### 7. Risk Assessment

*Risk Matrix*
- Crawl blocked (rate limit, anti-bot): High impact, medium probability
- LLM API downtime: High impact, low probability
- AWS budget overrun: Medium impact, low probability

*Mitigation Strategies*
- Crawl: Follow `ROBOTSTXT_OBEY`, `DOWNLOAD_DELAY`, valid User-Agent
- LLM: Fallback pattern (Qwen3 → Llama 3.1 → Gemini Flash)
- Cost: AWS Budget Alerts, Fargate spot instances when possible

### 8. Expected Outcomes

*Technical Improvements*:
- Automated pipeline crawling + processing ~500 articles/day from 3 sources
- Cost reduced ~30% compared to v1 (~$35 → ~$21-26)
- Vietnamese news-based RAG Q&A system

*Long-term Value*:
- Expandable news data platform
- Real-world experience with AWS serverless architecture
- Reusable for other RAG use cases