---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Building a News RAG Pipeline System on AWS

#### Overview

**News RAG** is an intelligent news aggregation and analysis system leveraging RAG (Retrieval-Augmented Generation) architecture to automatically collect news from Vietnamese newspapers, process data using Star Schema, create embedding vectors, and enable users to ask questions in natural language.

The system is fully deployed on AWS with serverless/container architecture, using services: **ECS Fargate**, **SQS**, **ECR**, **RDS Aurora PostgreSQL**, **EventBridge**, **CloudWatch**, integrated with **Qdrant Cloud** (vector DB), **Groq/Gemini** (LLM).

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
                       (Embed → Qdrant)

                              ▲
                              │ top-k chunks
                     RAG API ◄── API Gateway ◄── Frontend (Next.js)
```

#### Workshop Contents

1. [System Architecture Overview](5.1-Workshop-overview/)
2. [Development Environment Setup](5.2-Prerequiste/)
3. [Crawler Module — Scrapy Spider on ECS Fargate](5.3-S3-vpc/)
4. [ETL & Vectorize Module — Data Processing and Embedding](5.4-S3-onprem/)
5. [RAG API Module — Intelligent Q&A](5.5-Policy/)
6. [Deployment and Operations with Terraform](5.6-Cleanup/)