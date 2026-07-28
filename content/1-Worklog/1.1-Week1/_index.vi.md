---
title: "Worklog Tuần 1"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu tuần 1:

* Làm quen với chương trình First Cloud AI Journey (FCAJ) và các thành viên trong nhóm.
* Tìm hiểu các dịch vụ AWS cơ bản liên quan đến dự án.
* Đọc hiểu kiến trúc tổng thể của dự án NewsRAG (phiên bản khóa luận gốc — v1).
* Clone repository và thiết lập môi trường phát triển local.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                                   | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Làm quen với các thành viên FCAJ <br> - Đọc nội quy, quy định của chương trình thực tập <br> - Nhận nhiệm vụ và phân chia nhóm dự án                                                     | 16/06/2026   | 16/06/2026      |                                           |
| 3   | - Tìm hiểu tổng quan AWS và các nhóm dịch vụ: <br>&emsp; + Compute (EC2, ECS, Fargate, Lambda) <br>&emsp; + Storage (S3) <br>&emsp; + Database (RDS, Aurora) <br>&emsp; + Networking (VPC, SG) <br>&emsp; + Messaging (SQS, Kafka) | 17/06/2026   | 17/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 4   | - Tạo AWS Free Tier account <br> - Cài đặt AWS CLI & cấu hình credentials <br> - Tìm hiểu AWS Management Console                                                                          | 18/06/2026   | 18/06/2026      | <https://cloudjourney.awsstudygroup.com/> |
| 5   | - Đọc tài liệu kiến trúc NewsRAG v1 (khóa luận gốc) <br> - Tìm hiểu kiến trúc RAG (Retrieval-Augmented Generation) <br> - Nghiên cứu các thành phần: Crawler, Kafka, ETL, Vector DB, LLM  | 19/06/2026   | 19/06/2026      | Pipeline_v3.md                            |
| 6   | - Clone repository NewsRAG về máy <br> - Cài đặt Python virtual environment <br> - Chạy `docker-compose up -d` để khởi tạo PostgreSQL + Kafka local <br> - Đọc hiểu cấu trúc source code    | 20/06/2026   | 20/06/2026      | README.md, Makefile                       |


### Kết quả đạt được tuần 1:

* Hiểu được tổng quan về AWS và các nhóm dịch vụ liên quan đến dự án NewsRAG:
  * **Compute**: EC2, ECS Fargate (chạy container), Lambda (serverless functions)
  * **Database**: RDS Aurora PostgreSQL (lưu trữ bài viết + vector search)
  * **Messaging**: SQS (hàng đợi tin nhắn), Kafka (streaming — dùng trong v1)
  * **Container**: ECR (registry), ECS (orchestration)

* Đã tạo và cấu hình AWS Free Tier account thành công, bao gồm:
  * Access Key / Secret Key
  * Region mặc định: `ap-southeast-2`
  * AWS CLI hoạt động trên terminal

* Hiểu được kiến trúc tổng thể NewsRAG:
  * **v1 (Khóa luận)**: Lambda Crawler + Kafka + BAAI/bge-m3 + Qdrant Cloud + FastAPI/Next.js
  * **v2 (Cải tiến)**: Fargate Crawler + SQS + Bedrock Titan Embed + Aurora pgvector
  * Phân chia module: Crawl+Queue (A), Consumer+DB (B), ETL+Embed (C), RAG+Frontend (D)

* Clone repo thành công, thiết lập môi trường local:
  * Python 3.10 virtual environment
  * PostgreSQL + Kafka chạy qua Docker Compose
  * Đọc hiểu cấu trúc thư mục: `crawler/`, `consumer/`, `etl/`, `vectorize/`, `search/`, `database/`
