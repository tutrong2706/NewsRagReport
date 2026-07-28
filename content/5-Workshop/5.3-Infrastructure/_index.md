---
title: "Infrastructure as Code (Terraform)"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3 </b> "
---

# Infrastructure as Code (Terraform)

This section covers the complete AWS infrastructure defined in `main.tf` using Terraform for the News RAG Pipeline.

## Architecture Overview

The Terraform configuration provisions:

- **VPC & Networking**: Custom VPC, public subnets in 2 AZs, Internet Gateway, Route Tables
- **Security**: Security Groups for ECS tasks and RDS, IAM Roles for ECS execution and tasks
- **Database**: Aurora Serverless v2 PostgreSQL with pgvector extension
- **Container Registry**: ECR repository for Docker images
- **Container Orchestration**: ECS Cluster with Fargate task definitions
- **Scheduling**: EventBridge Scheduler rules for daily pipeline execution
- **Observability**: CloudWatch Log Groups

## File Structure

```
AWS-Projects/
├── main.tf              # Main Terraform configuration
├── terraform.tfvars     # Sensitive variables (gitignored)
├── .env.example         # Environment variable template
└── database/
    └── warehouse.sql    # Star Schema + pgvector DDL
```

## Provider Configuration

```hcl
provider "aws" {
  region = "ap-southeast-2"  # Sydney - supports Bedrock Titan Embed v2
}
```

> **Region Note:** Ensure `amazon.titan-embed-text-v2:0` is available in your region. Check [Bedrock Model Regions](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html).

## Variables (terraform.tfvars)

```hcl
# Database
db_password     = "your_secure_password_123!"
db_user         = "postgres"

# Qdrant (not used in AWS v2, but kept for compatibility)
qdrant_api_key  = ""
qdrant_host     = ""

# LLM API Keys
model_1_api_key = "gsk_xxxxxxxxxxxxxxxxxxxx"  # Groq Qwen3
model_2_api_key = "gsk_xxxxxxxxxxxxxxxxxxxx"  # Groq Llama backup
model_3_api_key = "AIza_xxxxxxxxxxxxxxxxxxxx" # Gemini Flash
```

> **Security:** Never commit `terraform.tfvars`. It's in `.gitignore`.

## VPC & Networking (main.tf:40-80)

```hcl
# VPC with DNS hostnames enabled (required for RDS)
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "newsrag-vpc" }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

# Public Subnet AZ-a
resource "aws_subnet" "pub_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-southeast-2a"
  map_public_ip_on_launch = true
}

# Public Subnet AZ-b
resource "aws_subnet" "pub_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "ap-southeast-2b"
  map_public_ip_on_launch = true
}

# Route Table with IGW route
resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

# Associate subnets with route table
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.pub_a.id
  route_table_id = aws_route_table.rt.id
}
resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.pub_b.id
  route_table_id = aws_route_table.rt.id
}
```

**Why public subnets?** Fargate tasks with `assign_public_ip = true` need public subnets to:
1. Pull Docker images from ECR
2. Call Bedrock, Groq, Gemini APIs
3. Reach SQS, CloudWatch endpoints

## Security Groups (main.tf:82-114)

```hcl
# ECS Tasks Security Group - Outbound only (Internet access)
resource "aws_security_group" "ecs_sg" {
  name   = "newsrag-ecs-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# RDS Security Group - Inbound from ECS SG only on port 5432
resource "aws_security_group" "rds_sg" {
  name   = "newsrag-rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Key Design:** ECS tasks can reach Internet (for APIs, ECR). RDS only accepts connections from ECS security group on port 5432.

## RDS Aurora Serverless v2 (main.tf:116-144)

```hcl
# DB Subnet Group (requires subnets in 2+ AZs)
resource "aws_db_subnet_group" "main" {
  name       = "newsrag-db-subnet"
  subnet_ids = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
}

# Aurora Cluster
resource "aws_rds_cluster" "main" {
  cluster_identifier     = "newsrag-postgres"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"  # Supports pgvector
  database_name          = "newsrag"
  master_username        = var.db_user
  master_password        = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  skip_final_snapshot    = true  # WARNING: Data loss on destroy
}

# Aurora Instance (Serverless v2)
resource "aws_rds_cluster_instance" "main" {
  identifier         = "newsrag-postgres-1"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.t4g.medium"  # 2 vCPU, 4 GB RAM
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version
}
```

**Aurora Serverless v2 Benefits:**
- Auto-scales capacity (0.5 - 128 ACUs)
- Pay per second for actual usage
- `db.t4g.medium` = 2 ACU baseline (~$0.12/hr = ~$87/month at 100% usage)
- Typical usage ~2 ACU = ~$15-20/month

## ECR Repository (main.tf:147)

```hcl
resource "aws_ecr_repository" "api" {
  name = "newsrag-api"
}
```

## ECS Cluster (main.tf:150)

```hcl
resource "aws_ecs_cluster" "cluster" {
  name = "newsrag-cluster"
}
```

## IAM Roles (main.tf:152-180)

```hcl
# ECS Task Execution Role (pull images, write CloudWatch logs)
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "newsrag-ecs-execution-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Role (permissions for application: Bedrock, SQS, Secrets Manager)
resource "aws_iam_role" "ecs_task_role" {
  name = "newsrag-ecs-task-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

# Attach custom policy for Bedrock, SQS, etc. (create separately or inline)
```

> **Note:** The Task Role needs inline policy for `bedrock:InvokeModel`, `sqs:*`, `secretsmanager:GetSecretValue`. See [IAM Policy for Lambda](5.12-Lambda-ETL/#iam-policy-for-lambda) section.

## CloudWatch Log Group (main.tf:183-186)

```hcl
resource "aws_cloudwatch_log_group" "logs" {
  name              = "/ecs/newsrag-project"
  retention_in_days = 7
}
```

## Environment Variables (main.tf:189-218)

All task definitions share these environment variables:

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
    { name = "KAFKA_BOOTSTRAP_SERVERS", value = "localhost:9092" },  # Legacy
    { name = "KAFKA_TOPIC_NEWS", value = "news_raw" },               # Legacy
    { name = "EMBEDDING_MODEL", value = "BAAI/bge-small-en-v1.5" },  # Legacy
    { name = "EMBEDDING_SIZE", value = "384" },                      # Legacy
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

> **Note:** Kafka/Qdrant/local embedding vars are kept for compatibility but not used in AWS v2 architecture.

## ECS Task Definitions (main.tf:222-292)

### Crawler Task (main.tf:222-244)
```hcl
resource "aws_ecs_task_definition" "crawler" {
  family                   = "newsrag-crawler"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"      # 0.25 vCPU
  memory                   = "512"      # 0.5 GB
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn
  container_definitions = jsonencode([{
    name  = "crawler"
    image = "${aws_ecr_repository.api.repository_url}:latest"
    command = ["python", "main.py", "--mode", "crawl"]
    environment = local.common_env
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.logs.name
        "awslogs-region"        = "ap-southeast-2"
        "awslogs-stream-prefix" = "crawler"
      }
    }
  }])
}
```

### ETL Task (main.tf:246-268)
```hcl
resource "aws_ecs_task_definition" "etl" {
  family                   = "newsrag-etl"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"      # 0.5 vCPU
  memory                   = "1024"     # 1 GB
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn
  container_definitions = jsonencode([{
    name  = "etl"
    image = "${aws_ecr_repository.api.repository_url}:latest"
    command = ["python", "main.py", "--mode", "etl"]
    environment = local.common_env
    logConfiguration = { ... }
  }])
}
```

### Vectorize Task (main.tf:270-292)
```hcl
resource "aws_ecs_task_definition" "vectorize" {
  family                   = "newsrag-vectorize"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn
  container_definitions = jsonencode([{
    name  = "vectorize"
    image = "${aws_ecr_repository.api.repository_url}:latest"
    command = ["python", "main.py", "--mode", "vectorize"]
    environment = local.common_env
    logConfiguration = { ... }
  }])
}
```

> **AWS v2 Note:** In production v2, the vectorize task is merged into ETL Lambda. This task definition is kept for reference/local compatibility.

## EventBridge Scheduler (main.tf:294-357)

Daily cron jobs (UTC timezone):

```hcl
# Crawler: 01:00 UTC daily
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

# ETL: 02:00 UTC daily
resource "aws_cloudwatch_event_rule" "etl_schedule" {
  name                = "newsrag-etl-rule"
  schedule_expression = "cron(0 2 * * ? *)"
}
# ... etl_target similar to crawler_target

# Vectorize: 03:00 UTC daily (legacy)
resource "aws_cloudwatch_event_rule" "vectorize_schedule" {
  name                = "newsrag-vectorize-rule"
  schedule_expression = "cron(0 3 * * ? *)"
}
# ... vectorize_target
```

**Cron Expression Format:** `cron(minutes hours day-of-month month day-of-week year)`
- `cron(0 1 * * ? *)` = Daily at 01:00 UTC
- `?` = no specific day-of-week (use day-of-month instead)

## Outputs (main.tf:360-370)

```hcl
output "rds_endpoint" {
  value = aws_rds_cluster.main.endpoint
}

output "ecr_repository_url" {
  value = aws_ecr_repository.api.repository_url
}

output "ecs_cluster_name" {
  value = aws_ecs_cluster.cluster.name
}
```

## Deployment Commands

```bash
# 1. Initialize
terraform init

# 2. Plan (review changes)
terraform plan

# 3. Apply
terraform apply
# Type 'yes' to confirm

# 4. Get outputs
terraform output rds_endpoint
terraform output ecr_repository_url
terraform output ecs_cluster_name
```

## Post-Deployment: Initialize Database

```bash
# Get RDS endpoint
ENDPOINT=$(terraform output -raw rds_endpoint)

# Run schema
psql "postgresql://postgres:your_password@${ENDPOINT}:5432/newsrag" -f database/warehouse.sql
```

The `warehouse.sql` creates:
- **Dimension Tables**: `dim_source`, `dim_time`, `dim_content`, `dim_author`
- **Fact Tables**: `fact_articles`, `fact_article_authors`, `fact_chunks`
- **pgvector Extension + HNSW Index** on `fact_chunks.embedding vector(1024)`

## Verification

```bash
# Check ECS Cluster
aws ecs describe-clusters --clusters newsrag-cluster

# Check Task Definitions
aws ecs list-task-definitions --family-prefix newsrag

# Check EventBridge Rules
aws events list-rules --name-prefix newsrag

# Check RDS
aws rds describe-db-clusters --db-cluster-identifier newsrag-postgres

# View CloudWatch Logs
aws logs tail /ecs/newsrag-project --follow
```

## Cost Estimation (Monthly, ap-southeast-2)

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

## Cleanup

```bash
terraform destroy
# Type 'yes' to confirm
```

> ⚠️ **WARNING:** This destroys the Aurora cluster and **all data permanently**. Backup first if needed.

---

**Next:** [Local Development Setup](5.4-Local-Dev/)