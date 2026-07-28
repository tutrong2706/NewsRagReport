---
title: "News RAG Pipeline on AWS — Workshop"
date: 2026-07-28
weight: 5
chapter: false
pre: " <b> 5. </b> "
---
# News RAG Pipeline on AWS — Workshop

## Giới thiệu dự án

**News RAG Pipeline on AWS** là hệ thống end-to-end hoàn chỉnh tự động thu thập tin tức từ các báo điện tử Việt Nam, xử lý qua Data Warehouse (Star Schema), tạo vector embedding bằng Amazon Bedrock, và cung cấp giao diện hỏi đáp thông minh sử dụng kiến trúc RAG (Retrieval-Augmented Generation) — tất cả chạy serverless trên AWS.

## Tổng quan kiến trúc

### Phát triển cục bộ (v1 - Docker Compose)
```
Scrapy Crawler → Kafka → PostgreSQL → ETL (Star Schema) → SentenceTransformer → Qdrant → RAG API
```

### Triển khai Production AWS Serverless (v2 - Hiện tại)
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

## Các thành phần chính

| Thành phần | Công nghệ | Mục đích |
|------------|-----------|----------|
| **Scheduler** | EventBridge Scheduler | Cron job hàng ngày (01:00, 02:00 UTC) |
| **Crawler** | ECS Fargate + Scrapy SitemapSpider | Crawl sitemap báo, đẩy vào SQS |
| **Hàng đợi** | Amazon SQS Standard | Thay thế Kafka, ~$0/tháng |
| **Consumer** | AWS Lambda (SQS trigger) | SHA256 URL dedup, insert bài thô |
| **ETL + Embed** | AWS Lambda (timeout 15 phút) | Clean HTML → Chunk 500 token → Bedrock Titan Embed v2 → pgvector |
| **Kho dữ liệu** | Aurora Serverless v2 + pgvector | PostgreSQL + HNSW vector search, 2 ACU |
| **RAG API** | Lambda + API Gateway | Embed query → Tìm kiếm vector → LLM generate |
| **LLM** | Groq (Qwen3-8B) + Gemini 2.0 Flash | API bên ngoài, không host local |
| **Frontend** | Next.js + FastAPI | Dashboard, Search, Chat, Explorer, Monitor |

## Mục tiêu học tập

Sau khi hoàn thành workshop này, bạn sẽ có thể:

1. **Infrastructure as Code**: Định nghĩa hạ tầng AWS bằng Terraform (VPC, Aurora, ECS, Lambda, EventBridge)
2. **Serverless Containers**: Đóng gói và triển khai Scrapy crawler trên ECS Fargate với ECR
3. **Thiết kế Data Warehouse**: Triển khai Star Schema trên Aurora PostgreSQL với extension pgvector
4. **Tự động hóa luồng công việc**: Lên lịch pipeline hàng ngày bằng EventBridge Scheduler
5. **Hệ thống RAG**: Tích hợp vector database (pgvector) với API LLM cho hỏi đáp thông minh
6. **Tối ưu chi phí**: Tận dụng dịch vụ serverless để giảm thiểu chi phí vận hành (~$21-26/tháng)

## Các phần Workshop

### Giai đoạn 1: Nền tảng & Hạ tầng
1. [Tổng quan Workshop](5.1-Workshop-overview/) - Kiến trúc, mục tiêu, điều kiện tiên quyết, ước lượng chi phí
2. [Điều kiện tiên quyết](5.2-Prerequisites/) - AWS CLI, Terraform, Docker, Python, Git setup
3. [Hạ tầng dưới dạng Code (Terraform)](5.3-Infrastructure/) - VPC, Aurora pgvector, ECS, ECR, IAM, EventBridge

### Giai đoạn 2: Phát triển cục bộ
4. [Thiết lập phát triển cục bộ](5.4-Local-Dev/) - Docker Compose (PostgreSQL, Qdrant, Kafka)
5. [Phát triển Crawler](5.5-Crawler/) - Scrapy SitemapSpider cho các trang tin tức Việt Nam
6. [Nhập dữ liệu](5.6-Ingestion/) - Kafka Consumer → PostgreSQL lưu trữ thô
7. [ETL & Star Schema](5.7-ETL/) - Làm sạch dữ liệu, chunking, chuyển đổi Star Schema
8. [Vector hóa](5.8-Vectorize/) - SentenceTransformer embeddings → Qdrant

### Giai đoạn 3: Triển khai AWS
9. [Chuẩn bị triển khai AWS](5.9-AWS-Deploy/) - Docker build, ECR push, ECS Task Definitions
10. [Fargate Crawler](5.10-Fargate-Crawler/) - EventBridge → ECS RunTask → SQS
11. [Lambda Consumer](5.11-Lambda-Consumer/) - SQS Trigger → SHA256 Dedup → Aurora Insert
12. [Lambda ETL + Embedding](5.12-Lambda-ETL/) - Clean → Chunk → Bedrock Titan Embed → pgvector

### Giai đoạn 4: Ứng dụng RAG
13. [RAG API](5.13-RAG-API/) - Lambda + API Gateway → Bedrock Embed Query → pgvector Search → LLM
14. [Tích hợp Frontend](5.14-Frontend/) - Next.js Dashboard, Search, Chat, Explorer, Monitor

### Giai đoạn 5: Sẵn sàng Production
15. [Kiểm thử & Giám sát](5.15-Testing/) - Đánh giá RAGAS, CloudWatch dashboards, alerts
16. [Tối ưu chi phí](5.16-Cost/) - Serverless tuning, Fargate Spot, right-sizing
17. [Dọn dẹp](5.17-Cleanup/) - Terraform destroy, kiểm tra dọn dẹp thủ công

## Cấu trúc dự án

```
AWS-Projects/
├── config/
│   └── config_site.json          # Cấu hình các trang tin tức
├── crawler/
│   ├── spiders/spider.py         # Scrapy SitemapSpider
│   ├── pipelines.py              # SQS Pipeline
│   └── settings.py               # Cài đặt Scrapy
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
│   └── vectorize.py              # Vector hóa cục bộ (Legacy)
├── main.py                       # Entry Point (crawl/etl/vectorize/full)
├── main.tf                       # Terraform Infrastructure
├── Dockerfile                    # Multi-stage cho ECS Fargate
├── docker-compose.yml            # Môi trường Dev Local
├── deploy.sh                     # Script triển khai Lambda
├── requirements.txt              # Python Dependencies
└── README.md                     # Tài liệu dự án
```

## So sánh kiến trúc: v1 vs v2

| Thành phần | v1 (Khóa luận) | v2 (Workshop này) | Lý do |
|------------|----------------|-------------------|-------|
| **Crawler** | Lambda + Scrapy (timeout 15 phút) | **Fargate + SitemapSpider** | Không timeout, crawl được bài cũ qua sitemap |
| **Stream** | Kafka trên Docker | **SQS Standard (~$0)** | Đơn giản hơn, serverless, tiết kiệm chi phí |
| **Embedding** | Local BGE models | **Bedrock Titan Embed v2** | AWS native, serverless, vector space nhất quán |
| **Vector DB** | Qdrant Cloud | **Aurora + pgvector** | DB thống nhất, không phụ thuộc bên ngoài |
| **Vectorize** | Fargate Task riêng biệt | **Gộp vào ETL Lambda** | Ít dịch vụ hơn, pipeline đơn giản hơn |
| **RAG Query Embed** | Local model (cold start 5-10s) | **Bedrock API** | Embedding nhất quán, không cold start |
| **Chi phí/tháng** | ~$35 | **~$21-26** | Giảm ~30% |

> **Nguyên tắc quan trọng:** ETL và RAG API **bắt buộc dùng cùng một embedding model** (`amazon.titan-embed-text-v2:0`). Khác model = khác vector space = tìm kiếm hỏng.

## Ước lượng chi phí (Hàng tháng, ap-southeast-2)

| Dịch vụ | Cấu hình | Chi phí ước tính |
|---------|----------|------------------|
| Aurora Serverless v2 | 2 ACU (db.t4g.medium) | ~$15-20 |
| ECS Fargate Crawler | 0.25 vCPU, 0.5 GB, 30 phút/ngày | ~$0.50 |
| Lambda (Consumer, ETL, RAG) | ~1M invocations, timeout 15 phút | ~$2-3 |
| SQS Standard | ~30K messages/tháng | ~$0 |
| API Gateway | ~10K requests/tháng | ~$0.30 |
| Bedrock Titan Embed | ~500K tokens/tháng | ~$0.50 |
| CloudWatch Logs | Giữ 7 ngày | ~$1-2 |
| **Tổng cộng** | | **~$21-26/tháng** |

> Chi phí thay đổi theo region và mức sử dụng. Kiểm tra [AWS Pricing Calculator](https://calculator.aws/) cho mức giá hiện tại.

## Điều kiện tiên quyết

- **Tài khoản AWS** với quyền cho: VPC, EC2, RDS, ECS, ECR, Lambda, SQS, EventBridge, API Gateway, Bedrock, CloudWatch, IAM
- **AWS CLI** đã cấu hình (`aws configure`)
- **Terraform** >= 1.5.0
- **Docker** & **Docker Compose**
- **Python** 3.10+
- **Git**
- **API Keys bên ngoài**: Groq API Key, Google Gemini API Key
- **Truy cập Model Bedrock**: Bật `amazon.titan-embed-text-v2:0` trong AWS Console

## Khởi động nhanh

```bash
# 1. Clone & Setup
git clone <repo-url> AWS-Projects
cd AWS-Projects

# 2. Phát triển cục bộ
make setup
docker compose up -d
python main.py --mode full

# 3. Triển khai AWS
cp .env.example .env
# Chỉnh sửa .env với giá trị của bạn
cp terraform.tfvars.example terraform.tfvars
# Chỉnh sửa terraform.tfvars
terraform init
terraform apply

# 4. Build & Push Docker
docker build -t news-crawler .
aws ecr get-login-password | docker login --username AWS --password-stdin <ECR_URL>
docker tag news-crawler:latest <ECR_URL>/news-crawler:latest
docker push <ECR_URL>/news-crawler:latest

# 5. Triển khai Lambdas
./deploy.sh

# 6. Test RAG API
curl -X POST <API_URL>/ask -d '{"query": "Tóm tắt tin kinh tế hôm nay"}'
```

---

**Tiếp theo:** [Tổng quan Workshop](5.1-Workshop-overview/)