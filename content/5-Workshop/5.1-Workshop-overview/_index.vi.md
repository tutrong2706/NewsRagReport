---
title: "Tổng quan Workshop"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 5.1 </b> "
---


# News RAG Pipeline trên AWS — Tổng quan Workshop

Workshop này hướng dẫn bạn xây dựng một **News RAG (Retrieval-Augmented Generation) Pipeline** hoàn chỉnh trên AWS sử dụng kiến trúc serverless. Bạn sẽ học cách thiết kế, triển khai và vận hành một hệ thống end-to-end tự động thu thập tin tức, xử lý thành Data Warehouse, tạo vector embedding, và cho phép hỏi đáp thông minh bằng LLM — tất cả chạy serverless trên AWS.

## Kiến trúc Workshop

![Kiến trúc Workshop](/images/5-Workshop/5.1-Workshop-overview/image.png)
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

## Mục tiêu học tập

Sau khi hoàn thành workshop này, bạn sẽ có thể:

1. **Infrastructure as Code**: Định nghĩa hạ tầng AWS bằng Terraform (VPC, Aurora, ECS, Lambda, EventBridge)
2. **Serverless Containers**: Đóng gói và triển khai Scrapy crawler trên ECS Fargate với ECR
3. **Thiết kế Data Warehouse**: Triển khai Star Schema trên Aurora PostgreSQL với extension pgvector
4. **Tự động hóa Workflow**: Lên lịch pipeline hàng ngày bằng EventBridge Scheduler
5. **Hệ thống RAG**: Tích hợp vector database (pgvector) với LLM APIs cho hỏi đáp thông minh
6. **Tối ưu chi phí**: Tận dụng dịch vụ serverless để giảm thiểu chi phí vận hành (~$21-26/tháng)

## Các Module Workshop

| Module | Tiêu đề                                | Mô tả                                                        | Thời gian |
| --------| ----------------------------------------| --------------------------------------------------------------| -----------|
| 5.1    | **Tổng quan Workshop**                 | Kiến trúc, mục tiêu, điều kiện tiên quyết, ước lượng chi phí | 30 phút   |
| 5.2    | **Điều kiện tiên quyết**               | AWS CLI, Terraform, Docker, Python setup                     | 30 phút   |
| 5.3    | **Hạ tầng (Terraform)**                | VPC, Aurora pgvector, ECS Cluster, ECR, IAM, EventBridge     | 60 phút   |
| 5.4    | **Crawler (ECS Fargate)**              | Dockerfile, Scrapy SitemapSpider, ECR push, Task Definition  | 45 phút   |
| 5.5    | **Hàng đợi & Consumer (SQS + Lambda)** | SQS queue, Lambda Consumer, SHA256 dedup, Aurora insert      | 45 phút   |
| 5.6    | **ETL + Embedding (Lambda + Bedrock)** | HTML clean, chunking, Titan Embed v2, pgvector HNSW insert   | 60 phút   |
| 5.7    | **RAG API (Lambda + API Gateway)**     | Query embed, pgvector search, LLM generation, response       | 45 phút   |
| 5.8    | **Tích hợp Frontend**                  | Next.js Dashboard, Search, Chat, Explorer, Pipeline Monitor  | 60 phút   |
| 5.9    | **Kiểm thử & Giám sát**                | CloudWatch Logs, query testing, RAG evaluation               | 30 phút   |
| 5.10   | **Dọn dẹp**                            | Terraform destroy, dọn dẹp tài nguyên                        | 15 phút   |

## Điều kiện tiên quyết

- **Tài khoản AWS** với quyền cho: VPC, EC2, RDS, ECS, ECR, Lambda, SQS, EventBridge, API Gateway, Bedrock, CloudWatch, IAM
- **AWS CLI** đã cấu hình (`aws configure`) với thông tin xác thực phù hợp
- **Terraform** >= 1.5.0 đã cài đặt
- **Docker** & **Docker Compose** đã cài đặt
- **Python** 3.10+ với pip
- **Git** để kiểm soát phiên bản
- **Trình soạn thảo mã** (VS Code khuyến nghị)

## Ước lượng chi phí (Hàng tháng, ap-southeast-2)

| Dịch vụ | Cấu hình | Chi phí ước tính |
|---------|---------------|-------------------|
| Aurora Serverless v2 | 2 ACU (db.t4g.medium) | ~$15-20 |
| ECS Fargate Crawler | 0.25 vCPU, 0.5 GB, 30 phút/ngày | ~$0.50 |
| Lambda (Consumer, ETL, RAG) | ~1M invocations, timeout 15 phút | ~$2-3 |
| SQS Standard | ~30K messages/tháng | ~$0 |
| API Gateway | ~10K requests/tháng | ~$0.30 |
| Bedrock Titan Embed | ~500K tokens/tháng | ~$0.50 |
| CloudWatch Logs | Giữ 7 ngày | ~$1-2 |
| **Tổng cộng** | | **~$21-26/tháng** |

> **Lưu ý:** Chi phí thay đổi theo khu vực và mức sử dụng. Kiểm tra [AWS Pricing Calculator](https://calculator.aws/) để biết mức giá hiện tại.

## Cấu trúc dự án

```
AWS-Projects/
├── config/
│   └── config_site.json          # Cấu hình các trang báo
├── crawler/
│   ├── spiders/spider.py         # Scrapy SitemapSpider
│   ├── pipelines.py              # SQS pipeline
│   └── settings.py               # Cài đặt Scrapy
├── consumer/
│   └── consumer.py               # Lambda Consumer (SQS → Aurora)
├── etl/
│   └── etl_warehouse.py          # ETL + Bedrock Embedding
├── search/
│   ├── engine.py                 # RAG Pipeline
│   ├── retriever.py              # Tìm kiếm pgvector
│   ├── generator.py              # LLM generation (Groq/Gemini)
│   └── schemas.py                # Models Pydantic
├── database/
│   └── warehouse.sql             # Star Schema + pgvector
├── vectorize/
│   └── vectorize.py              # Vector hóa cũ (legacy)
├── main.py                       # Entry point (crawl/etl/vectorize/full)
├── main.tf                       # Terraform infrastructure
├── Dockerfile                    # Multi-stage cho ECS Fargate
├── docker-compose.yml            # Dev local (PostgreSQL, Qdrant, Kafka)
├── deploy.sh                     # Script deploy Lambda
├── requirements.txt              # Python dependencies
└── README.md                     # Tài liệu dự án
```

## So sánh Kiến trúc: v1 vs v2

| Thành phần | v1 (Khóa luận) | v2 (Workshop này) | Lý do |
|------------|----------------|-------------------|-------|
| **Crawler** | Lambda + Scrapy (timeout 15 phút) | **Fargate + SitemapSpider** | Không timeout, crawl được bài cũ qua sitemap |
| **Stream** | Kafka trên Docker | **SQS Standard (~$0)** | Đơn giản hơn, serverless, tiết kiệm chi phí |
| **Embedding** | Model BGE cục bộ | **Bedrock Titan Embed v2** | AWS native, serverless, vector space nhất quán |
| **Vector DB** | Qdrant Cloud | **Aurora + pgvector** | DB thống nhất, không phụ thuộc bên ngoài |
| **Vectorize** | Fargate task riêng | **Gộp vào ETL Lambda** | Giảm dịch vụ, pipeline đơn giản hơn |
| **RAG Query Embed** | Model local (cold start 5-10s) | **Bedrock API** | Nhất quán embedding, không cold start |
| **Chi phí/tháng** | ~$35 | **~$21-26** | Giảm ~30% |

> **Nguyên tắc quan trọng:** ETL và RAG API **bắt buộc dùng cùng một embedding model** (`amazon.titan-embed-text-v2:0`). Model khác nhau = vector space khác nhau = tìm kiếm hỏng.

---

**Tiếp theo:** [Điều kiện tiên quyết](5.2-Prerequisites/)