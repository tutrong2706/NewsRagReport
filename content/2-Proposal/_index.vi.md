---
title: "Bản đề xuất"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 2. </b> "
---


# News RAG Pipeline trên AWS
## Đường ống dữ liệu Serverless với RAG cho hỏi đáp tin tức thông minh

### 1. Tóm tắt điều hành
News RAG Pipeline là hệ thống xây dựng ứng dụng hỏi đáp tin tức thông minh, tự động thu thập bài báo từ các trang tin Việt Nam (VnExpress, Thanh Niên, VietnamNet), xử lý qua Data Warehouse (Star Schema), tạo vector embedding bằng Amazon Bedrock Titan Embed v2, và cho phép người dùng đặt câu hỏi bằng ngôn ngữ tự nhiên thông qua kiến trúc RAG (Retrieval-Augmented Generation) — hoàn toàn serverless trên AWS. Hệ thống là minh chứng thực tế về tích hợp dịch vụ serverless AWS với LLM API (Groq, Gemini) để tạo nền tảng tra cứu thông tin AI.

### 2. Tuyên bố vấn đề
#### Vấn đề hiện tại
Theo dõi tin tức từ nhiều nguồn đòi hỏi đọc và tìm kiếm thủ công qua vô số bài báo. Không có hệ thống tập trung cho phép người dùng đặt câu hỏi về tin tức mới nhất và nhận câu trả lời AI có trích dẫn nguồn. Các giải pháp hiện có như ChatGPT thiếu ngữ cảnh tin tức cập nhật.

#### Giải pháp
News RAG Pipeline tự động hóa toàn bộ quy trình: (1) Scrapy crawler trên ECS Fargate thu thập bài báo từ sitemap tin tức, (2) bài viết được đưa qua SQS vào Aurora PostgreSQL, (3) Lambda ETL xử lý HTML thô thành Star Schema và tạo vector embedding qua Amazon Bedrock Titan Embed v2, (4) Lambda RAG API nhận câu hỏi ngôn ngữ tự nhiên, tìm kiếm tương đồng vector trên pgvector (HNSW index), và sinh câu trả lời dùng Groq (Qwen3-8B, Llama 3.1) hoặc Gemini 2.0 Flash — tất cả được điều phối bởi EventBridge Scheduler.

#### Lợi ích và hoàn vốn đầu tư
Giải pháp cung cấp nền tảng học tập về kiến trúc serverless AWS, hệ thống RAG và MLOps. Lợi ích chính: tổng hợp tin tức tự động loại bỏ tìm kiếm thủ công, hỏi đáp AI có trích dẫn nguồn tiết kiệm thời gian nghiên cứu, trải nghiệm thực tế với 10+ dịch vụ AWS, và hạ tầng serverless tiết kiệm chi phí. Chi phí hàng tháng khoảng $21-26 USD.

### 3. Kiến trúc giải pháp
Nền tảng sử dụng kiến trúc serverless AWS với hai giai đoạn pipeline: (1) Data Pipeline — EventBridge Scheduler kích hoạt ECS Fargate crawler hàng ngày lúc 01:00 UTC, đẩy bài viết vào SQS; Lambda Consumer xử lý messages và insert bài thô vào Aurora PostgreSQL với SHA256 dedup. (2) ETL + RAG Pipeline — EventBridge thứ hai lúc 02:00 UTC chạy Lambda ETL làm sạch HTML, chunk text 500 token, tạo embedding 1024 chiều qua Bedrock Titan Embed v2, lưu vector vào Aurora pgvector với HNSW index. Lambda RAG API, phía sau API Gateway, embed câu hỏi người dùng với Bedrock, tìm kiếm tương đồng trên pgvector, sau đó sinh câu trả lời dùng Groq/Gemini LLM.

![News RAG Pipeline Architecture](images/image.png)

### Các dịch vụ AWS sử dụng
- **Amazon ECS Fargate**: Chạy Scrapy SitemapSpider crawler (0.25 vCPU, 0.5 GB)
- **Amazon SQS Standard**: Message queue thay thế Kafka (~$0/tháng)
- **AWS Lambda** (3 functions): Consumer (SQS → Aurora), ETL + Bedrock Embed, RAG API
- **Amazon Aurora Serverless v2**: PostgreSQL 15.4 với pgvector extension
- **Amazon Bedrock**: Titan Embed Text v2 (1024d) cho vector embeddings
- **Amazon API Gateway**: REST API frontend cho RAG queries
- **Amazon EventBridge Scheduler**: Cron trigger hàng ngày (01:00, 02:00 UTC)
- **Amazon ECR**: Docker image registry cho Fargate tasks
- **AWS IAM**: Task execution và Lambda roles với least-privilege policies
- **Amazon CloudWatch**: Centralized logging và monitoring (7-day retention)

### Thiết kế thành phần
- **Crawler (Fargate)**: Scrapy SitemapSpider đọc sitemap_news.xml từ 3 nguồn báo, parse bài viết, đẩy vào SQS. Chạy ~30 phút/ngày.
- **Queue (SQS)**: Standard queue với DLQ, 14-day retention, 3 lần retry.
- **Consumer (Lambda)**: Triggered bởi SQS, tính SHA256 URL hash cho dedup, insert bài thô vào Aurora.
- **ETL + Embed (Lambda)**: Làm sạch HTML, chunk text (500 tokens, 50 overlap), gọi Bedrock Titan Embed v2, lưu vector 1024d trong pgvector với HNSW index.
- **RAG API (Lambda + API Gateway)**: Embed câu hỏi qua Bedrock, cosine similarity search trên pgvector, sinh câu trả lời qua Groq/Gemini có trích dẫn nguồn.
- **Frontend (Next.js + FastAPI)**: Dashboard với KPI cards, charts (Recharts), AI Chat với model selector, Article Explorer, Pipeline Monitor.

### 4. Triển khai kỹ thuật
#### Các giai đoạn triển khai
Dự án theo 4 giai đoạn:
- **Giai đoạn 1 - Hạ tầng (Tuần 1-2)**: Terraform cho VPC, Aurora pgvector, ECS Cluster, ECR, Lambda, EventBridge, IAM, CloudWatch. Xây dựng Docker multi-stage cho Fargate.
- **Giai đoạn 2 - Phát triển cục bộ (Tuần 3-4)**: Docker Compose với PostgreSQL, Qdrant, Kafka. Phát triển Scrapy SitemapSpider, Kafka Consumer, ETL pipeline với Star Schema, SentenceTransformer vectorization.
- **Giai đoạn 3 - Production AWS (Tuần 5-6)**: Deploy Fargate crawler với EventBridge scheduler, Lambda Consumer với SQS trigger, Lambda ETL với Bedrock Titan Embed v2, Lambda RAG API với API Gateway, Next.js frontend.
- **Giai đoạn 4 - Kiểm thử (Tuần 7-8)**: RAGAS evaluation (Faithfulness, Relevancy, Precision, Recall), CloudWatch dashboards và alerts, Locust load testing, tối ưu chi phí.

#### Yêu cầu kỹ thuật
- **Data Pipeline**: Scrapy với SitemapSpider, Kafka (local) / SQS (AWS), PostgreSQL với pgvector.
- **ETL Pipeline**: Làm sạch HTML bằng regex, chunk text 500 tokens, embedding qua SentenceTransformer (local) / Bedrock Titan v2 (AWS).
- **RAG System**: pgvector HNSW similarity search (cosine distance), Groq API (Qwen3-8B, Llama 3.1), Gemini 2.0 Flash fallback, structured prompts với trích dẫn nguồn.
- **Infrastructure**: Terraform cho AWS resources, Docker multi-stage builds, Lambda deployment packages, EventBridge cron scheduling.

### 5. Timeline & Milestones
**Dòng thời gian dự án**
- Pre-Internship (Tháng 0): Lập kế hoạch, học AWS cơ bản.
- Internship (Tháng 1-3):
  - Tháng 1: Thiết lập hạ tầng, môi trường dev local, phát triển crawler.
  - Tháng 2: ETL pipeline, Star Schema, vector hóa, phát triển RAG API.
  - Tháng 3: Triển khai AWS production, kiểm thử (RAGAS), monitoring, tối ưu chi phí.
- Post-Launch: Duy trì và mở rộng (semantic chunking, hybrid search, topic alerts).

### 6. Dự toán ngân sách
Tham khảo [AWS Pricing Calculator](https://calculator.aws/#/) để ước tính.

#### Chi phí hạ tầng (Hàng tháng)
- Aurora Serverless v2 (2 ACU): ~$15-20
- ECS Fargate Crawler (0.25 vCPU, 0.5 GB, 30 phút/ngày): ~$0.50
- Lambda (3 functions): ~$2-3
- SQS Standard: ~$0
- API Gateway: ~$0.30
- Bedrock Titan Embed: ~$0.50
- CloudWatch Logs (7-day retention): ~$1-2
- **Tổng: ~$21-26/tháng**

### 7. Đánh giá rủi ro
#### Ma trận rủi ro
- Crawler bị chặn: Medium impact, medium probability (giảm thiểu với headers phù hợp, tôn trọng robots.txt, DOWNLOAD_DELAY).
- Bedrock throttling: Medium impact, low probability (giảm thiểu với retry logic, adaptive backoff).
- LLM API outage (Groq/Gemini): High impact, low probability (giảm thiểu với multi-model fallback).
- Vượt chi phí: Medium impact, low probability (giảm thiểu với budget alerts, right-sizing).

#### Chiến lược giảm thiểu
- **Crawler**: Tôn trọng robots.txt, 1s DOWNLOAD_DELAY, AutoThrottle bật.
- **Throttling**: Exponential backoff trên Bedrock, max_attempts=3.
- **LLM Fallback**: Chuỗi fallback: Groq Qwen3 → Llama 3.1 → Gemini 2.0 Flash.
- **Chi phí**: AWS Budget alerts tại 80%, Fargate Spot cho crawler, right-size Lambda memory.

#### Kế hoạch dự phòng
- Nếu site báo chặn crawler: Dùng nguồn tin thay thế hoặc upload dữ liệu thủ công.
- Nếu Bedrock không available: Dùng SentenceTransformer local.
- Nếu chi phí vượt ngân sách: Giảm Aurora xuống 1 ACU.

### 8. Kết quả mong đợi
#### Cải thiện kỹ thuật:
Tổng hợp tin tức tự động thay thế tìm kiếm thủ công. Hỏi đáp AI có trích dẫn nguồn tiết kiệm thời gian. Kiến trúc serverless co giãn xử lý khối lượng tin tức tăng dần.

#### Giá trị dài hạn
- Hệ thống RAG nền tảng cho các dự án NLP/AI trong tương lai.
- Các thành phần pipeline có thể tái sử dụng cho lĩnh vực khác (tech blogs, research papers).
- Kinh nghiệm thực tế với AWS serverless.
- Nền tảng dữ liệu (~5000 bài viết) cho phân tích sau này.
