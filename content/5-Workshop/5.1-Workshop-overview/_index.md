---
title : "Overview"
date : 2024-01-01 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### NewsRAG Architecture Overview

The NewsRAG system is designed as a pipeline architecture with 5 main modules:

1. **Crawler** (ECS Fargate): Scrapy Spider crawls news from 3 newspapers via sitemap XML
2. **Consumer** (Lambda): Receives messages from SQS, deduplicates URLs with SHA256, inserts into PostgreSQL
3. **ETL** (Fargate): Cleans HTML, chunks text at 800 chars, transforms into Star Schema
4. **Vectorize** (Fargate): Embeds using BAAI/bge-small-en-v1.5, upserts to Qdrant Cloud
5. **RAG API** (Lambda + API Gateway): Embeds query → vector search → LLM generates answer

#### Technologies Used

| Component       | Technology                              |
|----------------|----------------------------------------|
| Crawler         | Python, Scrapy, newspaper3k            |
| Queue           | Amazon SQS (replaced Kafka from v1)    |
| Database        | Aurora PostgreSQL + pgvector           |
| ETL             | Python, RecursiveCharacterTextSplitter |
| Embedding       | BAAI/bge-small-en-v1.5 (384d)         |
| Vector DB       | Qdrant Cloud                           |
| LLM             | Groq (Qwen3-8B), Gemini 2.0 Flash     |
| Frontend        | Next.js + React + Tailwind CSS         |
| Infrastructure  | Terraform, Docker, ECS Fargate         |
| Monitoring      | CloudWatch Logs                        |

#### Team Module Distribution

| Member     | Module              | Main Responsibilities                          |
| ---------- | ------------------- | ---------------------------------------------- |
| **A (Me)** | Crawl + Queue       | Scrapy Spider, Dockerfile, SQS, ECR, Fargate   |
| **B**      | Consumer + Database | Lambda Consumer, Aurora, Star Schema SQL        |
| **C**      | ETL + Embedding     | Clean text, chunk, embedding, insert vector     |
| **D**      | RAG API + Frontend  | RAG search, API Gateway, Next.js Dashboard      |