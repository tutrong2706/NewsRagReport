---
title: "Blog 3"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Triển khai Infrastructure as Code với Terraform cho Dự án AWS

Trong dự án NewsRAG, toàn bộ infrastructure AWS được quản lý bằng **Terraform** — công cụ Infrastructure as Code (IaC) phổ biến nhất. Bài viết chia sẻ cách tổ chức Terraform config cho một dự án thực tế với VPC, RDS, ECS, EventBridge.

### Tại sao Infrastructure as Code?

- **Reproducible**: Tạo lại toàn bộ infrastructure bằng 1 lệnh `terraform apply`
- **Version control**: Track thay đổi qua Git
- **Tự động hóa**: Không cần click thủ công trên AWS Console
- **Team collaboration**: Review infrastructure changes qua Pull Request

### Cấu trúc Terraform cho NewsRAG

```
main.tf (371 lines)
├── Provider: aws (ap-southeast-2)
├── Variables: db_password, qdrant_api_key, model_api_keys (sensitive)
├── VPC & Networking
│   ├── VPC (10.0.0.0/16)
│   ├── 2 Public Subnets (2a, 2b)
│   ├── Internet Gateway + Route Table
│   └── Security Groups (ECS, RDS)
├── RDS Aurora PostgreSQL
│   ├── Cluster: aurora-postgresql 15.4
│   └── Instance: db.t4g.medium
├── ECS + ECR
│   ├── ECR Repository
│   ├── ECS Cluster
│   ├── 3 Task Definitions (crawler, etl, vectorize)
│   └── IAM Roles
├── EventBridge Scheduler
│   ├── Crawler: 01:00 UTC
│   ├── ETL: 02:00 UTC
│   └── Vectorize: 03:00 UTC
└── Outputs: rds_endpoint, ecr_url, cluster_name
```

### Best Practices áp dụng

1. **Sensitive variables**: Dùng `type = string, sensitive = true` cho passwords và API keys
2. **Locals block**: Quản lý environment variables chung qua `locals.common_env`
3. **Security Groups**: Principle of least privilege — RDS chỉ accept từ ECS SG
4. **CloudWatch Logs**: Retention 7 ngày để tiết kiệm chi phí
5. **Outputs**: Export RDS endpoint, ECR URL để dùng trong CI/CD

### Kết quả

- 1 file `main.tf` quản lý toàn bộ 20+ AWS resources
- Deploy/destroy infrastructure trong < 10 phút
- Dễ dàng tái tạo môi trường mới

...Hình ảnh...

...Link bài blog...