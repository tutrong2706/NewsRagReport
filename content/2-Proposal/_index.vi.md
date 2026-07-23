---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# News RAG — Hệ thống Tổng hợp và Phân tích Tin tức Thông minh
## Giải pháp AWS Serverless cho tổng hợp tin tức với kiến trúc RAG

### 1. Tóm tắt điều hành

**News RAG** là hệ thống tổng hợp và phân tích tin tức thông minh, ứng dụng kiến trúc RAG (Retrieval-Augmented Generation) để tự động thu thập tin tức từ các báo điện tử Việt Nam (VnExpress, Thanh Niên, VietnamNet), xử lý và lưu trữ dữ liệu theo mô hình Star Schema, tạo embedding vector và cho phép người dùng đặt câu hỏi bằng ngôn ngữ tự nhiên — hệ thống sẽ truy xuất các đoạn tin tức liên quan và tổng hợp câu trả lời thông qua các mô hình ngôn ngữ lớn (LLM) như Groq Qwen3, Gemini Flash.

Dự án được phát triển bởi nhóm 4 thành viên trong chương trình thực tập **First Cloud AI Journey** tại **AWS Vietnam**, triển khai hoàn toàn trên AWS với kiến trúc serverless, tối ưu chi phí (~$21-26/tháng).

### 2. Tuyên bố vấn đề

*Vấn đề hiện tại*
Người dùng Việt Nam gặp khó khăn trong việc theo dõi và phân tích thông tin từ nhiều nguồn báo khác nhau. Việc đọc thủ công hàng trăm bài báo mỗi ngày tốn rất nhiều thời gian. Chưa có hệ thống nào cho phép hỏi đáp thông minh dựa trên dữ liệu tin tức tiếng Việt sử dụng công nghệ RAG.

*Giải pháp*
Hệ thống NewsRAG tự động:
1. **Crawl** tin tức từ 3 báo điện tử qua Scrapy SitemapSpider trên ECS Fargate
2. **Stream** dữ liệu qua SQS → Lambda Consumer → PostgreSQL
3. **ETL** chuẩn hóa dữ liệu theo Star Schema, chunk text 800 chars
4. **Vectorize** embedding qua Bedrock Titan Embed v2 (1024 chiều)
5. **RAG** cho phép hỏi đáp thông minh: embed query → vector search → LLM generate

*Lợi ích*
- Tiết kiệm thời gian theo dõi tin tức cho người dùng
- Cung cấp trả lời có trích dẫn nguồn, tăng độ tin cậy
- Chi phí vận hành thấp (~$21-26/tháng) nhờ kiến trúc serverless
- Có thể mở rộng thêm nguồn tin và ngôn ngữ

### 3. Kiến trúc giải pháp

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

*Dịch vụ AWS sử dụng*
- **ECS Fargate**: Chạy container Crawler, ETL, Vectorize (không quản lý server)
- **Amazon SQS**: Hàng đợi tin nhắn thay Kafka (~$0/tháng)
- **Amazon ECR**: Docker image registry
- **RDS Aurora PostgreSQL**: Lưu trữ bài viết + Star Schema warehouse
- **Amazon Bedrock**: Titan Embed Text v2 (1024 chiều) — serverless embedding
- **EventBridge Scheduler**: Tự động chạy pipeline theo lịch
- **CloudWatch**: Monitoring logs
- **VPC + Security Groups**: Networking an toàn

*Dịch vụ bên ngoài*
- **Qdrant Cloud**: Vector database cho similarity search
- **Groq API** (Qwen3-8B): LLM chính cho RAG
- **Google Gemini 2.0 Flash**: LLM fallback

### 4. Triển khai kỹ thuật

*Phân chia công việc theo nhóm*

| Thành viên | Module              | AWS Services                                 | Công việc chính                                          |
| ---------- | ------------------- | -------------------------------------------- | -------------------------------------------------------- |
| **A (Tôi)**| Crawl + Queue       | ECS Fargate, SQS, ECR, Docker                | Dockerfile, SQS + DLQ, ECR deploy, Sitemap Spider        |
| **B**      | Consumer + Database | Lambda, Aurora, pgvector, Secrets Manager    | Lambda Consumer, Aurora cluster, Star Schema SQL          |
| **C**      | ETL + Embedding     | Lambda, Bedrock, S3                          | Clean HTML, chunk text, Bedrock Embed, insert vector      |
| **D**      | RAG API + Frontend  | Lambda, API Gateway, Groq/Gemini, Next.js    | RAG search, API Gateway, Frontend Dashboard               |

*Công nghệ sử dụng*
- **Backend**: Python 3.10, Scrapy, psycopg2, boto3, sentence-transformers
- **Frontend**: Next.js, React, Tailwind CSS, FastAPI
- **Infrastructure**: Terraform, Docker, Docker Compose
- **Database**: PostgreSQL 15 + pgvector extension
- **LLM**: LangChain + Groq + Google GenAI

### 5. Lộ trình & Timeline (8 tuần)

| Tuần    | Công việc chính                                          |
| ------- | -------------------------------------------------------- |
| **1**   | Làm quen AWS, tìm hiểu kiến trúc NewsRAG, setup môi trường |
| **2**   | Nghiên cứu Scrapy, Docker, phân tích sitemap XML        |
| **3**   | Phát triển NewsRAGSpider, KafkaPipeline                  |
| **4**   | Dockerize Crawler, push ECR, cấu hình ECS Fargate       |
| **5**   | SQS + DLQ, tích hợp Crawler → Consumer → PostgreSQL     |
| **6**   | Terraform infrastructure, EventBridge Schedule           |
| **7**   | Integration test, debug edge cases, đánh giá Ragas       |
| **8**   | Hoàn thiện code, viết tài liệu, báo cáo thực tập       |

### 6. Ước tính chi phí

| Nhóm              | Component                                              | Giá/tháng         |
| ----------------- | ------------------------------------------------------ | ----------------- |
| **Crawl + Queue** | Fargate (0.25 vCPU, 512 MB, ~30 phút/ngày) + SQS + Consumer | **~$3–5**     |
| **ETL + Embed**   | Fargate ETL + Bedrock Titan Embed                      | **~$1–3**         |
| **Database**      | Aurora Serverless v2 (2 ACU) + pgvector                | **~$14**          |
| **RAG API**       | Lambda RAG + API Gateway                               | **~$3–4**         |
| **Tổng**          |                                                        | **~$21–26/tháng** |

### 7. Đánh giá rủi ro

*Ma trận rủi ro*
- Crawl bị chặn (rate limit, anti-bot): Ảnh hưởng cao, xác suất trung bình
- API LLM ngừng hoạt động: Ảnh hưởng cao, xác suất thấp
- Vượt ngân sách AWS: Ảnh hưởng trung bình, xác suất thấp

*Chiến lược giảm thiểu*
- Crawl: Tuân thủ `ROBOTSTXT_OBEY`, `DOWNLOAD_DELAY`, User-Agent hợp lệ
- LLM: Fallback pattern (Qwen3 → Llama 3.1 → Gemini Flash)
- Chi phí: AWS Budget Alerts, Fargate spot instances khi có thể

### 8. Kết quả kỳ vọng

*Cải tiến kỹ thuật*:
- Pipeline tự động crawl + xử lý ~500 bài/ngày từ 3 nguồn báo
- Chi phí giảm ~30% so với phiên bản v1 (~$35 → ~$21-26)
- Hệ thống RAG hỏi đáp tiếng Việt dựa trên tin tức thực

*Giá trị dài hạn*:
- Nền tảng dữ liệu tin tức có thể mở rộng thêm nguồn
- Kinh nghiệm thực tế với AWS serverless architecture
- Có thể tái sử dụng cho các use case RAG khác