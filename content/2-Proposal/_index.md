---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---



# News RAG Pipeline on AWS
## A Serverless Retrieval-Augmented Generation Pipeline for Intelligent News Q&A

### 1. Executive Summary
The News RAG Pipeline is designed to build an intelligent news Q&A system that automatically crawls news articles from Vietnamese news sites (VnExpress, Thanh Nien, VietnamNet), processes them through a Data Warehouse (Star Schema), generates vector embeddings using Amazon Bedrock Titan Embed v2, and enables natural language querying through Retrieval-Augmented Generation (RAG) — entirely serverless on AWS. The system serves as a practical demonstration of integrating AWS serverless services with LLM APIs (Groq, Gemini) to create an AI-powered information retrieval platform.

### 2. Problem Statement
#### What's the Problem?
Keeping up with news from multiple sources requires manually reading and searching through countless articles. There is no centralized system that allows users to ask questions about recent news and get AI-generated answers with proper citations. Existing solutions like ChatGPT lack up-to-date news context without manual prompting.

#### The Solution
The News RAG Pipeline automates the entire workflow: (1) Scrapy crawlers on ECS Fargate collect articles from Vietnamese news sitemaps, (2) articles are ingested via SQS into Aurora PostgreSQL, (3) Lambda ETL processes raw HTML into a Star Schema and generates vector embeddings via Amazon Bedrock Titan Embed v2, (4) a Lambda RAG API accepts natural language queries, performs vector similarity search on pgvector (HNSW index), and generates answers using Groq (Qwen3-8B, Llama 3.1) or Gemini 2.0 Flash LLMs — all orchestrated by EventBridge Scheduler.

#### Benefits and Return on Investment
The solution provides a foundational platform for learning modern AWS serverless architecture, RAG systems, and MLOps practices. Key benefits include: automated news aggregation eliminates manual searching, AI-powered Q&A with source citations saves research time, hands-on experience with 10+ AWS services, and cost-effective serverless infrastructure. Monthly costs are approximately $21-26 USD. The break-even period is achieved through significant time savings from automated information retrieval versus manual news monitoring.

### 3. Solution Architecture
The platform employs a serverless AWS architecture based on two pipeline stages: (1) Data Pipeline — EventBridge Scheduler triggers ECS Fargate crawler daily at 01:00 UTC, pushing articles to SQS; Lambda Consumer processes messages and inserts raw articles into Aurora PostgreSQL with SHA256 deduplication. (2) ETL + RAG Pipeline — second EventBridge trigger at 02:00 UTC runs Lambda ETL that cleans HTML, chunks text at 500 tokens, generates 1024-dimensional embeddings via Bedrock Titan Embed v2, and stores vectors in Aurora pgvector with HNSW index. The RAG API Lambda, fronted by API Gateway, embeds user queries with Bedrock, performs similarity search on pgvector, then generates answers using Groq/Gemini LLMs. The architecture is detailed below:

![News RAG Pipeline Architecture](/images/5-Workshop/5.1-Workshop-overview/architecture.png)

### AWS Services Used
- **Amazon ECS Fargate**: Runs Scrapy SitemapSpider crawler (0.25 vCPU, 0.5 GB)
- **Amazon SQS Standard**: Message queue replacing Kafka (~$0/month)
- **AWS Lambda** (3 functions): Consumer (SQS → Aurora), ETL + Bedrock Embed, RAG API
- **Amazon Aurora Serverless v2**: PostgreSQL 15.4 with pgvector extension
- **Amazon Bedrock**: Titan Embed Text v2 (1024d) for vector embeddings
- **Amazon API Gateway**: REST API frontend for RAG queries
- **Amazon EventBridge Scheduler**: Daily cron triggers (01:00, 02:00 UTC)
- **Amazon ECR**: Docker image registry for Fargate tasks
- **AWS IAM**: Task execution and Lambda roles with least-privilege policies
- **Amazon CloudWatch**: Centralized logging and monitoring (7-day retention)

### Component Design
- **Crawler (Fargate)**: Scrapy SitemapSpider reads sitemap_news.xml from 3 news sources, parses articles, pushes to SQS. Runs daily ~30 min.
- **Queue (SQS)**: Standard queue with DLQ, 14-day retention, 3 retry limit.
- **Consumer (Lambda)**: Triggered by SQS, computes SHA256 URL hash for dedup, inserts raw articles into Aurora.
- **ETL + Embed (Lambda)**: Clean HTML, chunk text (500 tokens, 50 overlap), call Bedrock Titan Embed v2, store 1024d vectors in pgvector with HNSW index.
- **RAG API (Lambda + API Gateway)**: Embed user query via Bedrock, cosine similarity search on pgvector, generate answer via Groq/Gemini with source citations.
- **Frontend (Next.js + FastAPI)**: Dashboard with KPI cards, charts (Recharts), AI Chat with model selector, Article Explorer, Pipeline Monitor.

### 4. Technical Implementation
#### Implementation Phases
This project follows 4 phases:
- **Phase 1 - Infrastructure (Weeks 1-2)**: Terraform configuration for VPC, Aurora pgvector, ECS Cluster, ECR, Lambda, EventBridge, IAM, CloudWatch. Build multi-stage Docker image for Fargate.
- **Phase 2 - Local Development (Weeks 3-6)**: Docker Compose environment with PostgreSQL, Qdrant, Kafka. Develop Scrapy SitemapSpider, Kafka Consumer, ETL pipeline with Star Schema, SentenceTransformer vectorization.
- **Phase 3 - AWS Production (Weeks 7-10)**: Deploy Fargate crawler with EventBridge scheduler, Lambda Consumer with SQS trigger, Lambda ETL with Bedrock Titan Embed v2, Lambda RAG API with API Gateway, Next.js frontend.
- **Phase 4 - Testing & Polish (Weeks 11-12)**: RAGAS evaluation (Faithfulness, Relevancy, Precision, Recall), CloudWatch dashboards and alerts, Locust load testing, cost optimization.

#### Technical Requirements
- **Data Pipeline**: Scrapy with SitemapSpider for news crawling, Kafka (local) / SQS (AWS) for message queue, PostgreSQL with pgvector for storage and vector search.
- **ETL Pipeline**: HTML cleaning via regex, text chunking at 500 tokens with 50 overlap, embedding generation via SentenceTransformer (local) / Bedrock Titan v2 (AWS).
- **RAG System**: pgvector HNSW similarity search (cosine distance), Groq API (Qwen3-8B, Llama 3.1), Gemini 2.0 Flash fallback, structured prompts with source citation.
- **Infrastructure**: Terraform for all AWS resources, Docker multi-stage builds, Lambda deployment packages, EventBridge cron scheduling.

### 5. Timeline & Milestones
**Project Timeline**
- Pre-Internship (Month 0): Planning and AWS fundamentals study.
- Internship (Months 1-3):
  - Month 1: Infrastructure setup, local development environment, crawler development.
  - Month 2: ETL pipeline, Star Schema, vectorization, RAG API development.
  - Month 3: AWS production deployment, testing (RAGAS), monitoring, cost optimization.
- Post-Launch: Maintain and extend with additional features (semantic chunking, hybrid search, topic alerts).

### 6. Budget Estimation
You can use [AWS Pricing Calculator](https://calculator.aws/#/) for estimation.  

#### Infrastructure Costs (Monthly)
- Aurora Serverless v2 (2 ACU): ~$15-20
- ECS Fargate Crawler (0.25 vCPU, 0.5 GB, 30 min/day): ~$0.50
- Lambda (3 functions): ~$2-3
- SQS Standard: ~$0
- API Gateway: ~$0.30
- Bedrock Titan Embed: ~$0.50
- CloudWatch Logs (7-day retention): ~$1-2
- **Total: ~$21-26/month**

### 7. Risk Assessment
#### Risk Matrix
- Crawler Blocked by Websites: Medium impact, medium probability (mitigate with proper headers, respect robots.txt, DOWNLOAD_DELAY).
- Bedrock Throttling: Medium impact, low probability (mitigate with retry logic, adaptive backoff).
- LLM API Outages (Groq/Gemini): High impact, low probability (mitigate with multi-model fallback).
- Cost Overruns: Medium impact, low probability (mitigate with budget alerts, serverless right-sizing).

#### Mitigation Strategies
- **Crawler**: Respect robots.txt, 1s DOWNLOAD_DELAY, AutoThrottle enabled.
- **Throttling**: Exponential backoff on Bedrock calls, max_attempts=3.
- **LLM Fallback**: Fallback chain: Groq Qwen3 → Llama 3.1 → Gemini 2.0 Flash.
- **Cost**: AWS Budget alerts at 80%, Fargate Spot for crawler, right-size Lambda memory.

#### Contingency Plans
- If specific news sites block crawler: Use alternative news sources or manual data upload.
- If Bedrock unavailable in region: Use local SentenceTransformer model as fallback.
- If costs exceed budget: Reduce Aurora to 1 ACU, increase Lambda timeout instead of memory.

### 8. Expected Outcomes
#### Technical Improvements:
Automated news aggregation replaces manual searching. AI-powered Q&A with source citations saves research time. Scalable serverless architecture handles increasing news volume. 

#### Long-term Value
- Foundational RAG system for future NLP/AI projects.
- Reusable pipeline components for other domains (tech blogs, research papers).
- Hands-on AWS serverless expertise.
- Data foundation (~5000 articles accumulated over internship) for future analysis.
