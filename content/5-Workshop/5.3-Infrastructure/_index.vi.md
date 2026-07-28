---
title: "Hạ tầng dưới dạng Code (Terraform)"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3 </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Thông tin dưới đây chỉ mang tính tham khảo. Vui lòng **không sao chép y nguyên** cho báo cáo của bạn, bao gồm cả cảnh báo này.
{{% /notice %}}

# Hạ tầng dưới dạng Code (Terraform)

Phần này mô tả việc định nghĩa và triển khai toàn bộ hạ tầng AWS cho News RAG Pipeline bằng Terraform.

## Kiến trúc Terraform

File chính: `main.tf` — định nghĩa tất cả tài nguyên AWS:

```
VPC & Network → Security Groups → RDS Aurora → ECR → ECS Cluster → IAM Roles → CloudWatch Logs → Task Definitions → EventBridge Scheduler
```

## 1. VPC & Network (`main.tf:40-80`)

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "newsrag-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

# 2 Public Subnets across 2 AZs
resource "aws_subnet" "pub_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-southeast-2a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "pub_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "ap-southeast-2b"
  map_public_ip_on_launch = true
}

# Route Table → IGW
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.igw.id }
}
```

**Kết quả:** VPC `newsrag-vpc` với 2 public subnet ở 2 AZ, Internet Gateway, Route Table cho phép traffic ra Internet.

## 2. Security Groups (`main.tf:82-114`)

```hcl
# ECS Security Group — cho phép ra Internet (pull image, gọi Bedrock, Groq, Gemini)
resource "aws_security_group" "ecs_sg" {
  name   = "newsrag-ecs-sg"
  vpc_id = aws_vpc.main.id
  egress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}

# RDS Security Group — chỉ cho phép ECS truy cập port 5432
resource "aws_security_group" "rds_sg" {
  name   = "newsrag-rds-sg"
  vpc_id = aws_vpc.main.id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_sg.id]
  }
  egress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}
```

## 3. Aurora PostgreSQL Serverless v2 (`main.tf:116-144`)

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "newsrag-db-subnet"
  subnet_ids = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
}

resource "aws_rds_cluster" "main" {
  cluster_identifier     = "newsrag-postgres"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  database_name          = "newsrag"
  master_username        = var.db_user
  master_password        = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  skip_final_snapshot    = true  # ⚠️ Production: set false
}

resource "aws_rds_cluster_instance" "main" {
  identifier         = "newsrag-postgres-1"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.t4g.medium"  # 2 vCPU, 4GB RAM
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version
}
```

**Lưu ý quan trọng:**
- `db.t4g.medium` = 2 ACU (Aurora Capacity Units) ~ $15-20/tháng
- `skip_final_snapshot = true` chỉ dùng cho dev/test. Production nên bật snapshot.
- Sau khi deploy, chạy `database/warehouse.sql` để tạo schema Star Schema + pgvector.

## 4. ECR Repository (`main.tf:147`)

```hcl
resource "aws_ecr_repository" "api" { name = "newsrag-api" }
```
Chứa Docker image cho cả Crawler, ETL, và Vectorize tasks.

## 5. ECS Cluster (`main.tf:150`)

```hcl
resource "aws_ecs_cluster" "cluster" { name = "newsrag-cluster" }
```

## 6. IAM Roles (`main.tf:152-180`)

```hcl
# Execution Role — ECS pull image, ghi CloudWatch Logs
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "newsrag-ecs-execution-role"
  assume_role_policy = jsonencode({...})
}

resource "aws_iam_role_policy_attachment" "ecs_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Task Role — quyền gọi Bedrock, SQS, v.v.
resource "aws_iam_role" "ecs_task_role" {
  name = "newsrag-ecs-task-role"
  assume_role_policy = jsonencode({...})
}
# Gắn policy BedrockInvokeModel, SQSFullAccess, v.v. vào task role này
```

## 7. CloudWatch Log Group (`main.tf:183-186`)

```hcl
resource "aws_cloudwatch_log_group" "logs" {
  name              = "/ecs/newsrag-project"
  retention_in_days = 7
}
```

## 8. Biến môi trường chung (`main.tf:189-218`)

```hcl
locals {
  common_env = [
    { name = "DB_NAME", value = "newsrag" },
    { name = "DB_USER", value = var.db_user },
    { name = "DB_PASSWORD", value = var.db_password },
    { name = "DB_HOST", value = aws_rds_cluster.main.endpoint },
    { name = "DB_PORT", value = "5432" },
    { name = "QDRANT_HOST", value = var.qdrant_host },
    { name = "QDRANT_PORT", value = "6333" },
    { name = "QDRANT_API_KEY", value = var.qdrant_api_key },
    { name = "QDRANT_COLLECTION_NAME", value = "news_chunks" },
    { name = "KAFKA_BOOTSTRAP_SERVERS", value = "localhost:9092" },
    { name = "KAFKA_TOPIC_NEWS", value = "news_raw" },
    { name = "EMBEDDING_MODEL", value = "BAAI/bge-small-en-v1.5" },
    { name = "EMBEDDING_SIZE", value = "384" },
    { name = "NUM_MODEL_SUPPORT", value = "3" },
    { name = "MODEL_1_NAME", value = "qwen3-8b-instant" },
    { name = "MODEL_1_MODEL_ID", value = "qwen/qwen3-8b-instant" },
    { name = "MODEL_1_PROVIDER", value = "groq" },
    { name = "MODEL_1_API_KEY", value = var.model_1_api_key },
    { name = "MODEL_2_NAME", value = "llama-3.1-8b-instant" },
    { name = "MODEL_2_MODEL_ID", value = "meta-llama/llama-3.1-8b-instant" },
    { name = "MODEL_2_PROVIDER", value = "groq" },
    { name = "MODEL_2_API_KEY", value = var.model_2_api_key },
    { name = "MODEL_3_NAME", value = "gemini-2.0-flash" },
    { name = "MODEL_3_MODEL_ID", value = "gemini-2.0-flash" },
    { name = "MODEL_3_PROVIDER", value = "google" },
    { name = "MODEL_3_API_KEY", value = var.model_3_api_key },
  ]
}
```

> **Lưu ý:** Trên AWS production, `QDRANT_*` và `KAFKA_*` không dùng (thay bằng SQS + Aurora pgvector). Giữ lại cho tương thích code local.

## 9. ECS Task Definitions (`main.tf:222-292`)

Ba task definitions dùng chung Docker image:

| Task | CPU | Memory | Command |
|------|-----|--------|---------|
| **crawler** | 256 (0.25 vCPU) | 512 MB | `python main.py --mode crawl` |
| **etl** | 512 (0.5 vCPU) | 1024 MB | `python main.py --mode etl` |
| **vectorize** | 512 | 1024 MB | `python main.py --mode vectorize` |

Tất cả dùng:
- `network_mode = "awsvpc"` (bắt buộc cho Fargate)
- `execution_role_arn` = ECS execution role
- `task_role_arn` = ECS task role (Bedrock permissions)
- `awslogs` driver → `/ecs/newsrag-project`

## 10. EventBridge Scheduler (`main.tf:294-357`)

Cron jobs hàng ngày (UTC):

```hcl
# Crawler: 01:00 UTC hàng ngày
resource "aws_cloudwatch_event_rule" "crawler_schedule" {
  name                = "newsrag-crawler-rule"
  schedule_expression = "cron(0 1 * * ? *)"
}

resource "aws_cloudwatch_event_target" "crawler_target" {
  rule      = aws_cloudwatch_event_rule.crawler_schedule.name
  arn       = aws_ecs_cluster.cluster.arn
  role_arn  = aws_iam_role.ecs_task_execution_role.arn
  ecs_target {
    task_count          = 1
    task_definition_arn = aws_ecs_task_definition.crawler.arn
    launch_type         = "FARGATE"
    network_configuration {
      subnets          = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
      security_groups  = [aws_security_group.ecs_sg.id]
      assign_public_ip = true
    }
  }
}

# ETL: 02:00 UTC hàng ngày
resource "aws_cloudwatch_event_rule" "etl_schedule" {
  name                = "newsrag-etl-rule"
  schedule_expression = "cron(0 2 * * ? *)"
}
# ... etl_target tương tự

# Vectorize: 03:00 UTC hàng ngày (legacy)
resource "aws_cloudwatch_event_rule" "vectorize_schedule" {
  name                = "newsrag-vectorize-rule"
  schedule_expression = "cron(0 3 * * ? *)"
}
```

**Cron Expression Format:** `cron(minutes hours day-of-month month day-of-week year)`
- `cron(0 1 * * ? *)` = Hàng ngày lúc 01:00 UTC
- `?` = không có day-of-week cụ thể (dùng day-of-month thay thế)

## 11. Variables (`main.tf:5-38`)

```hcl
variable "db_password" { type = string; sensitive = true }
variable "db_user" { type = string }
variable "qdrant_api_key" { type = string; sensitive = true }
variable "qdrant_host" { type = string }
variable "model_1_api_key" { type = string; sensitive = true }
variable "model_2_api_key" { type = string; sensitive = true }
variable "model_3_api_key" { type = string; sensitive = true }
```

Điền vào `terraform.tfvars` (không commit lên git):
```hcl
db_password     = "your_secure_password_123!"
db_user         = "postgres"
qdrant_api_key  = ""
qdrant_host     = ""
model_1_api_key = "gsk_xxxxxxxxxxxxxxxxxxxx"  # Groq
model_2_api_key = "gsk_xxxxxxxxxxxxxxxxxxxx"  # Groq backup
model_3_api_key = "AIza_xxxxxxxxxxxxxxxxxxxx" # Gemini
```

## 12. Outputs (`main.tf:360-370`)

```hcl
output "rds_endpoint" { value = aws_rds_cluster.main.endpoint }
output "ecr_repository_url" { value = aws_ecr_repository.api.repository_url }
output "ecs_cluster_name" { value = aws_ecs_cluster.cluster.name }
```

Sau `terraform apply`, dùng outputs để:
- Kết nối database: `psql "postgresql://postgres:pass@$(terraform output -raw rds_endpoint):5432/newsrag"`
- Build & push Docker: `docker push $(terraform output -raw ecr_repository_url):latest`
- Chạy task thủ công: `aws ecs run-task --cluster $(terraform output -raw ecs_cluster_name) ...`

## Triển khai

```bash
# 1. Khởi tạo
terraform init

# 2. Xem kế hoạch
terraform plan

# 3. Áp dụng (nhập 'yes' khi được hỏi)
terraform apply

# 4. Lấy outputs
terraform output rds_endpoint
terraform output ecr_repository_url
terraform output ecs_cluster_name
```

## Khởi tạo Database Schema

```bash
ENDPOINT=$(terraform output -raw rds_endpoint)
psql "postgresql://postgres:your_password@${ENDPOINT}:5432/newsrag" -f database/warehouse.sql
```

File `database/warehouse.sql` tạo:
- **Dimension Tables**: `dim_source`, `dim_time`, `dim_content`, `dim_author`
- **Fact Tables**: `fact_articles`, `fact_article_authors`, `fact_chunks`
- **pgvector extension + HNSW index** trên `fact_chunks.embedding` (vector(1024))

## Xác minh triển khai

```bash
# Kiểm tra ECS Cluster
aws ecs describe-clusters --clusters newsrag-cluster

# Kiểm tra Task Definitions
aws ecs list-task-definitions --family-prefix newsrag

# Kiểm tra EventBridge Rules
aws events list-rules --name-prefix newsrag

# Kiểm tra RDS
aws rds describe-db-clusters --db-cluster-identifier newsrag-postgres

# Xem logs CloudWatch
aws logs tail /ecs/newsrag-project --follow
```

## Ước lượng chi phí (Hàng tháng, ap-southeast-2)

| Resource | Configuration | Est. Cost |
|----------|---------------|-----------|
| Aurora Serverless v2 | 2 ACU avg (db.t4g.medium) | ~$15-20 |
| ECS Fargate Crawler | 0.25 vCPU, 0.5 GB, 30 min/day | ~$0.50 |
| ECS Fargate ETL | 0.5 vCPU, 1 GB, 30 min/day | ~$1.00 |
| ECS Fargate Vectorize | 0.5 vCPU, 1 GB, 30 min/day | ~$1.00 |
| ECR | 1 repository, ~2 GB storage | ~$0.20 |
| CloudWatch Logs | 7-day retention, ~1 GB ingest | ~$1.00 |
| **Total (Infrastructure)** | | **~$19-24/month** |

> **Note:** Lambda + Bedrock + API Gateway costs (covered in later sections) add ~$2-5/month. **Total ~$21-29/month**.

## Dọn dẹp

```bash
terraform destroy
# Nhập 'yes' để xác nhận
```

> ⚠️ **CẢNH BÁO:** Lệnh này xóa Aurora cluster và **mất toàn bộ dữ liệu vĩnh viễn**. Backup trước nếu cần.

---

**Tiếp theo:** [Thiết lập phát triển cục bộ](5.4-Local-Dev/)