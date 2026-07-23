---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Xây dựng hệ thống News RAG Pipeline trên AWS

#### Tổng quan

**News RAG** là hệ thống tổng hợp và phân tích tin tức thông minh, ứng dụng kiến trúc RAG (Retrieval-Augmented Generation) để tự động thu thập tin tức từ các báo điện tử Việt Nam, xử lý dữ liệu theo mô hình Star Schema, tạo embedding vector, và cho phép người dùng hỏi đáp bằng ngôn ngữ tự nhiên.

Hệ thống được triển khai hoàn toàn trên AWS với kiến trúc serverless/container, sử dụng các dịch vụ: **ECS Fargate**, **SQS**, **ECR**, **RDS Aurora PostgreSQL**, **EventBridge**, **CloudWatch**, và tích hợp với **Qdrant Cloud** (vector DB), **Groq/Gemini** (LLM).

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

#### Nội dung workshop

1. [Tổng quan kiến trúc hệ thống](5.1-Workshop-overview/)
2. [Chuẩn bị môi trường phát triển](5.2-Prerequiste/)
3. [Module Crawler — Scrapy Spider trên ECS Fargate](5.3-S3-vpc/)
4. [Module ETL & Vectorize — Xử lý dữ liệu và embedding](5.4-S3-onprem/)
5. [Module RAG API — Hỏi đáp thông minh](5.5-Policy/)
6. [Triển khai và vận hành với Terraform](5.6-Cleanup/)