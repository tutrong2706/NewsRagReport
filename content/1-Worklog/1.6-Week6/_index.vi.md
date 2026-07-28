---
title: "Worklog Tuần 6"
date: 2026-07-28
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu tuần 6:

* Viết Terraform `main.tf` để provisioning toàn bộ infrastructure trên AWS.
* Cấu hình EventBridge Scheduler để tự động chạy pipeline theo lịch.
* Thiết lập IAM Roles, Security Groups, VPC cho ECS + RDS.
* Test terraform apply và kiểm tra tài nguyên trên AWS Console.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Viết Terraform — VPC & Networking: <br>&emsp; + VPC `newsrag-vpc` (CIDR 10.0.0.0/16) <br>&emsp; + 2 Public Subnets (ap-southeast-2a, 2b) <br>&emsp; + Internet Gateway + Route Table <br>&emsp; + Security Groups: `ecs_sg` (egress all), `rds_sg` (ingress 5432 from ECS) | 21/07/2026   | 21/07/2026      | main.tf                                   |
| 3   | - Viết Terraform — RDS Aurora PostgreSQL: <br>&emsp; + Cluster: `newsrag-postgres`, engine aurora-postgresql 15.4 <br>&emsp; + Instance: `db.t4g.medium` (2 vCPU, 4GB RAM) <br>&emsp; + DB Subnet Group cho multi-AZ | 22/07/2026   | 22/07/2026      | main.tf                                   |
| 4   | - Viết Terraform — ECS + ECR: <br>&emsp; + ECR Repository: `newsrag-api` <br>&emsp; + ECS Cluster: `newsrag-cluster` <br>&emsp; + 3 Task Definitions: crawler (256 CPU/512 MB), etl (512/1024), vectorize (512/1024) <br>&emsp; + IAM Roles: execution role + task role <br>&emsp; + CloudWatch Log Group: `/ecs/newsrag-project` | 23/07/2026   | 23/07/2026      | main.tf                                   |
| 5   | - Viết Terraform — EventBridge Scheduler: <br>&emsp; + Crawler: `cron(0 1 * * ? *)` — 01:00 UTC hàng ngày <br>&emsp; + ETL: `cron(0 2 * * ? *)` — 02:00 UTC <br>&emsp; + Vectorize: `cron(0 3 * * ? *)` — 03:00 UTC <br>&emsp; + Mỗi rule target tới ECS Fargate Task tương ứng | 24/07/2026   | 24/07/2026      | main.tf                                   |
| 6   | - `terraform init` & `terraform plan` — review changes <br> - `terraform apply` — deploy infrastructure <br> - Kiểm tra tài nguyên trên AWS Console <br> - Test EventBridge trigger manual | 25/07/2026   | 25/07/2026      |                                           |


### Kết quả đạt được tuần 6:

* Hoàn thành `main.tf` (371 dòng) quản lý toàn bộ infrastructure:
  * **VPC**: 1 VPC, 2 public subnets, Internet Gateway, Route Table
  * **Security Groups**: ECS (egress all), RDS (ingress 5432 chỉ từ ECS SG)
  * **RDS**: Aurora PostgreSQL 15.4, instance `db.t4g.medium`, skip final snapshot
  * **ECR**: Repository `newsrag-api`
  * **ECS**: Cluster + 3 Task Definitions (crawler, etl, vectorize)
  * **IAM**: Execution role (ECS Task Execution Policy) + Task role
  * **CloudWatch**: Log group `/ecs/newsrag-project`, retention 7 ngày
  * **EventBridge**: 3 scheduled rules

* Cấu hình EventBridge Scheduler:
  | Tên rule                | Cron expression        | Thời gian (UTC) | Target Task        |
  |------------------------|----------------------|-----------------|-------------------|
  | `newsrag-crawler-rule`  | `cron(0 1 * * ? *)`  | 01:00           | newsrag-crawler   |
  | `newsrag-etl-rule`      | `cron(0 2 * * ? *)`  | 02:00           | newsrag-etl       |
  | `newsrag-vectorize-rule`| `cron(0 3 * * ? *)`  | 03:00           | newsrag-vectorize |

* Environment variables được quản lý qua `locals.common_env`:
  * DB credentials → tự động lấy RDS endpoint
  * Qdrant, Kafka, Embedding, LLM configs
  * Sensitive values (passwords, API keys) → Terraform variables (type=string, sensitive=true)

* Outputs: `rds_endpoint`, `ecr_repository_url`, `ecs_cluster_name`

* `terraform apply` thành công, tất cả tài nguyên được tạo trên AWS
