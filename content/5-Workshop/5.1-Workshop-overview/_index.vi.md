---
title : "Giới thiệu"
date : 2024-01-01 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Tổng quan kiến trúc NewsRAG

Hệ thống NewsRAG được thiết kế theo kiến trúc pipeline gồm 5 module chính:

1. **Crawler** (ECS Fargate): Scrapy Spider crawl tin tức từ 3 báo qua sitemap XML
2. **Consumer** (Lambda): Nhận message từ SQS, dedup URL bằng SHA256, insert vào PostgreSQL
3. **ETL** (Fargate): Clean HTML, chunk text 800 chars, transform theo Star Schema
4. **Vectorize** (Fargate): Embedding bằng BAAI/bge-small-en-v1.5, upsert vào Qdrant Cloud
5. **RAG API** (Lambda + API Gateway): Embed query → vector search → LLM generate answer

#### Công nghệ sử dụng

| Component       | Công nghệ                              |
|----------------|----------------------------------------|
| Crawler         | Python, Scrapy, newspaper3k            |
| Queue           | Amazon SQS (thay Kafka từ v1)          |
| Database        | Aurora PostgreSQL + pgvector           |
| ETL             | Python, RecursiveCharacterTextSplitter |
| Embedding       | BAAI/bge-small-en-v1.5 (384d)         |
| Vector DB       | Qdrant Cloud                           |
| LLM             | Groq (Qwen3-8B), Gemini 2.0 Flash     |
| Frontend        | Next.js + React + Tailwind CSS         |
| Infrastructure  | Terraform, Docker, ECS Fargate         |
| Monitoring      | CloudWatch Logs                        |

#### Phân chia module theo nhóm

| Thành viên | Module              | Trách nhiệm chính                            |
| ---------- | ------------------- | --------------------------------------------- |
| **A (Tôi)**| Crawl + Queue       | Scrapy Spider, Dockerfile, SQS, ECR, Fargate  |
| **B**      | Consumer + Database | Lambda Consumer, Aurora, Star Schema SQL       |
| **C**      | ETL + Embedding     | Clean text, chunk, embedding, insert vector    |
| **D**      | RAG API + Frontend  | RAG search, API Gateway, Next.js Dashboard     |
