---
title : "Deploy"
date : 2024-01-01 
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

#### Triển khai Infrastructure as Code với Terraform

Thay vì cấu hình thủ công từng dịch vụ trên AWS Console, dự án NewsRAG sử dụng Terraform để tự động hóa việc khởi tạo toàn bộ hạ tầng (Infrastructure as Code - IaC).

**1. Cấu trúc Terraform**

File `main.tf` (hơn 370 dòng) quản lý tất cả các tài nguyên:
- **Networking**: VPC (`10.0.0.0/16`), 2 Public Subnets (multi-AZ), Internet Gateway, Route Tables.
- **Security Groups**: Phân quyền truy cập nguyên tắc đặc quyền tối thiểu (Least Privilege) — ví dụ RDS chỉ nhận traffic từ ECS Security Group.
- **Database**: Cụm RDS Aurora PostgreSQL Serverless.
- **Compute**: ECS Cluster, ECR Repository, và các Task Definitions cho Crawler, ETL, Vectorize chạy trên nền Fargate.
- **Scheduling**: EventBridge Rules để trigger các ECS Tasks theo cron schedule (01:00, 02:00, 03:00 UTC).

**2. Triển khai lên AWS**

Để triển khai hệ thống, bạn chỉ cần cấu hình AWS CLI và chạy các lệnh:

```bash
# Khởi tạo Terraform provider
terraform init

# Xem trước các tài nguyên sẽ được tạo
terraform plan

# Thực thi tạo tài nguyên trên AWS
terraform apply -auto-approve
```

**3. Tự động hóa Deployment (CI/CD)**

Script `deploy.sh` được cung cấp để tự động build Docker images, tag và push lên Amazon ECR, sau đó update ECS service để force deployment phiên bản mới nhất:

```bash
#!/bin/bash
# deploy.sh

# 1. Lấy thông tin xác thực ECR
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com

# 2. Build Docker images
docker build -t newsrag-crawler -f crawler/Dockerfile .
docker build -t newsrag-etl -f etl/Dockerfile .

# 3. Tag và Push lên ECR
docker tag newsrag-crawler:latest $ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/newsrag-api:crawler-latest
docker push $ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/newsrag-api:crawler-latest
```

**4. Dọn dẹp tài nguyên (Cleanup)**

Khi không sử dụng hệ thống để tránh phát sinh chi phí, bạn có thể xóa toàn bộ tài nguyên bằng 1 lệnh duy nhất:

```bash
terraform destroy
```

> **Lưu ý**: Lệnh này sẽ xóa tất cả VPC, ECS, RDS, EventBridge được định nghĩa trong `main.tf`. Dữ liệu trong database sẽ bị mất nếu bạn không cấu hình snapshot.