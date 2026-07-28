---
title: "Worklog Tuần 4"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Mục tiêu tuần 4:

* Đóng gói Crawler thành Docker container.
* Build và push Docker image lên Amazon ECR.
* Cấu hình ECS Task Definition cho Fargate Crawler.
* Test chạy Crawler trên Fargate thay vì local.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Viết Dockerfile cho Crawler: <br>&emsp; + Base image: `python:3.10-slim` <br>&emsp; + Cài đặt system deps: `build-essential`, `libpq-dev` <br>&emsp; + COPY requirements.txt & pip install <br>&emsp; + CMD: `python main.py --mode crawl` | 07/07/2025   | 07/07/2025      | Dockerfile                                |
| 3   | - Build Docker image local: `docker build -t news-crawler .` <br> - Test chạy container: `docker run news-crawler` <br> - Debug issues: PYTHONPATH, encoding, env variables              | 08/07/2025   | 08/07/2025      |                                           |
| 4   | - Tạo ECR repository: `newsrag-api` <br> - Push image lên ECR: <br>&emsp; + `aws ecr get-login-password` <br>&emsp; + `docker tag` & `docker push` <br> - Verify image trên AWS Console  | 09/07/2025   | 09/07/2025      | <https://docs.aws.amazon.com/ecr/>        |
| 5   | - Cấu hình ECS Task Definition cho Fargate: <br>&emsp; + CPU: 256 (0.25 vCPU), Memory: 512 MB <br>&emsp; + Network mode: awsvpc <br>&emsp; + Container command: `python main.py --mode crawl` <br>&emsp; + Environment variables từ `.env` <br>&emsp; + Log driver: awslogs → CloudWatch | 10/07/2025   | 10/07/2025      | main.tf                                   |
| 6   | - Tạo ECS Cluster: `newsrag-cluster` <br> - Test chạy Fargate Task manually <br> - Kiểm tra logs trên CloudWatch <br> - Debug networking issues (Security Groups, VPC, Public IP)        | 11/07/2025   | 11/07/2025      |                                           |


### Kết quả đạt được tuần 4:

* Hoàn thành Dockerfile tối ưu:
  ```dockerfile
  FROM python:3.10-slim
  WORKDIR /app
  RUN apt-get update && apt-get install -y build-essential libpq-dev curl
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY . .
  ENV PYTHONPATH=/app
  CMD ["python", "main.py", "--mode", "full"]
  ```

* Docker image build thành công, chạy ổn định trên local

* Push image lên ECR thành công:
  * Repository: `newsrag-api`
  * Tag: `latest`
  * Size: ~350 MB (sau tối ưu)

* Cấu hình ECS Task Definition hoàn chỉnh:
  * Family: `newsrag-crawler`
  * Fargate compatibility, CPU 256, Memory 512
  * awsvpc network mode cho private networking
  * CloudWatch Logs với prefix `crawler`
  * Environment variables inject từ Terraform locals

* ECS Cluster `newsrag-cluster` hoạt động, Fargate Task chạy thành công:
  * Crawler chạy trong container trên Fargate
  * Logs xuất hiện trên CloudWatch
  * Cần cấu hình Security Group cho phép egress (download pages, gọi Kafka)
