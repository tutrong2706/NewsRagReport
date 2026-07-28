---
title: "Tối ưu chi phí"
date: 2026-07-28
weight: 16
chapter: false
pre: " <b> 5.16 </b> "
---

# Tối ưu chi phí

Phần này bao gồm các chiến lược để giảm thiểu chi phí AWS cho News RAG Pipeline trong khi vẫn duy trì hiệu năng.

## Phân tích chi phí (Hàng tháng, ap-southeast-2)

| Dịch vụ | Cấu hình | Chi phí ước tính | % Tổng |
|---------|----------|------------------|--------|
| **Aurora Serverless v2** | 2 ACU baseline, auto-scale to 8 | $15-20 | ~50% |
| **ECS Fargate (Crawler)** | 0.25 vCPU, 0.5 GB, 30 min/day | $0.50 | ~1% |
| **ECS Fargate (ETL)** | 0.5 vCPU, 1 GB, 30 min/day | $1.00 | ~3% |
| **Lambda (Consumer)** | 512 MB, 1M req, 5s avg | $0.30 | ~1% |
| **Lambda (ETL)** | 1 GB, 30/day, 300s avg | $1.50 | ~4% |
| **Lambda (RAG API)** | 1 GB, 10K req, 3s avg | $0.50 | ~1% |
| **API Gateway** | 10K req/tháng | $0.35 | ~1% |
| **SQS** | 30K messages/tháng | $0.00 | ~0% |
| **Bedrock Titan Embed** | 500K tokens/day | $0.75 | ~2% |
| **ECR** | 2 GB storage | $0.20 | ~0% |
| **CloudWatch Logs** | 1 GB ingest, 7-day retention | $1.50 | ~4% |
| **Data Transfer** | < 1 GB/tháng | $0.09 | ~0% |
| **SNS (Alarms)** | < 100 notifications | $0.01 | ~0% |
| **Total** | | **~$21-26/tháng** | **100%** |

> **So sánh:** Kiến trúc v1 trước đây (Kafka, Qdrant Cloud, local embeddings) cost ~$35/tháng. **Tiết kiệm: ~30-40%**

## Chiến lược tối ưu

### 1. Aurora Serverless v2

```hcl
# Terraform: Set capacity range
resource "aws_rds_cluster" "main" {
  # ...
  serverlessv2_scaling_configuration {
    min_capacity = 0.5   # Minimum ACUs (có thể scale to 0 với pause)
    max_capacity = 8     # Maximum ACUs
  }
}

# Enable auto-pause cho dev environments
resource "aws_rds_cluster" "main" {
  # ...
  # For production, KHÔNG auto-pause (gây cold start)
  # For dev/staging:
  # engine_mode = "provisioned"  # hoặc dùng provisioned cho workload ổn định
}
```

**Tips:**
- Dùng **ACU = 0.5** minimum cho dev (scale to 0 khi idle)
- Production: **min=1, max=16** cho HA
- Monitor `ServerlessDatabaseCapacity` metric để tune
- Cân nhắc **Provisioned** với `db.t4g.medium` (~$45/tháng) nếu workload ổn định cao

### 2. Lambda Right-Sizing

```bash
# Dùng AWS Lambda Power Tuning để tìm optimal memory
# https://github.com/aws-lambda-power-tuning/aws-lambda-power-tuning

# Kết quả điển hình:
# - Consumer: 512 MB (CPU scales with memory)
# - ETL: 1024 MB (cần memory cho pgvector ops + Bedrock)
# - RAG API: 1024 MB (embedding + search + LLM)
```

**Cost vs Performance:**
| Memory | CPU | ETL Duration | Cost/Invocation |
|--------|-----|--------------|-----------------|
| 512 MB | 0.8 vCPU | 180s | $0.0000018 |
| 1024 MB | 1.7 vCPU | 90s | $0.0000018 |
| 2048 MB | 3.5 vCPU | 50s | $0.0000020 |

→ **1024 MB là sweet spot** cho ETL (2x nhanh hơn, same cost)

### 3. Fargate Spot cho Crawler

```hcl
# Thêm capacity provider strategy cho Spot
resource "aws_ecs_service" "crawler" {
  # ... (nếu chạy as service)
  # Cho scheduled tasks, dùng capacity provider trong RunTask override
}

# Trong EventBridge target:
ecs_target {
  launch_type = "FARGATE"
  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 4
    base              = 0
  }
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
    base              = 0
  }
}
```

**Tiết kiệm:** Lên đến 70% trên Fargate compute (~$0.15/tháng cho crawler)

### 4. Bedrock Embedding Optimization

```python
# Giảm chunk size, filter trước khi embed

# Current: 500 tokens/chunk, ~10 chunks/article
# Optimization: 300 tokens, 7 chunks → 30% fewer Bedrock calls

def chunk_text(text: str, chunk_size: int = 300, overlap: int = 30) -> list[str]:
    words = text.split()
    if len(words) <= chunk_size:
        return [text]
    
    chunks = []
    step = chunk_size - overlap
    for i in range(0, len(words), step):
        chunk = " ".join(words[i:i + chunk_size])
        if len(chunk.strip()) > 30:  # Skip tiny chunks
            chunks.append(chunk)
    return chunks
```

**Tiết kiệm:** ~$0.25/tháng (30% ít embeddings hơn)

### 5. CloudWatch Logs Retention

```hcl
resource "aws_cloudwatch_log_group" "logs" {
  name              = "/ecs/newsrag-project"
  retention_in_days = 3  # Giảm từ 7 cho dev; 14 cho prod
}
```

**Tiết kiệm:** 50% log cost ($0.75 vs $1.50/tháng)

### 6. S3 cho Cold Data (Optional)

```hcl
resource "aws_s3_bucket" "archive" {
  bucket = "newsrag-archive"
  lifecycle_rule {
    enabled = true
    transition {
      days          = 30
      storage_class = "GLACIER_IR"
    }
    transition {
      days          = 90
      storage_class = "DEEP_ARCHIVE"
    }
  }
}
```

Move `fact_chunks` cũ hơn 30 ngày lên S3 Glacier, chỉ giữ gần đây trong pgvector.

### 7. RDS Proxy (For High Concurrency)

```hcl
resource "aws_db_proxy" "main" {
  name                   = "newsrag-proxy"
  engine_family          = "POSTGRESQL"
  auth                   = [{ auth_scheme = "SECRETS", secret_arn = aws_secretsmanager_secret.db_credentials.arn, iam_auth = "DISABLED" }]
  role_arn               = aws_iam_role.rds_proxy_role.arn
  vpc_subnet_ids         = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
  vpc_security_group_ids = [aws_security_group.rds_sg.id]
  require_tls            = true
  idle_client_timeout    = 30
  max_connections_percent = 100
  max_idle_connections_percent = 50
}
```

**Khi dùng:** > 50 concurrent Lambda connections
**Cost:** ~$0.015/hr per vCPU = ~$11/tháng cho db.t4g.medium equivalent
**Lợi ích:** Connection pooling, reduced cold start, better scalability

## Cost Monitoring

### Budget Alert

```hcl
resource "aws_budgets_budget" "monthly" {
  name              = "newsrag-monthly-budget"
  budget_type       = "COST"
  limit_amount      = "30"
  limit_unit        = "USD"
  time_unit         = "MONTHLY"
  
  cost_filter = {
    name   = "LinkedAccount"
    values = [data.aws_caller_identity.current.account_id]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["your-email@example.com"]
  }
}
```

### Cost Explorer Queries

```bash
# Chi phí hàng ngày theo service (30 ngày qua)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Dự báo tháng này
aws ce get-cost-forecast \
  --time-period Start=$(date +%Y-%m-01),End=$(date -d '+1 month' +%Y-%m-01) \
  --metric BLENDED_COST \
  --granularity MONTHLY
```

## Cost Optimization Checklist

| ✅ | Optimization | Status | Savings |
|---|-------------|--------|---------|
| 1 | Aurora min ACU = 0.5 (dev) |  | $10/tháng |
| 2 | Lambda memory right-sized |  | $0.50/tháng |
| 3 | Fargate Spot cho crawler |  | $0.35/tháng |
| 4 | Chunk size 300 tokens |  | $0.25/tháng |
| 5 | CloudWatch logs 3-day retention (dev) |  | $0.75/tháng |
| 6 | Budget alert at 80% |  | Prevention |
| 7 | RDS Proxy (if >50 concurrent) |  | Scalability |
| 8 | S3 Glacier cho old data |  | Future |

**Total Potential Savings:** ~$12-15/tháng (50% reduction) cho dev environment

## Production vs Development Costs

| Environment | Aurora | Lambda | Fargate | Logs | Total |
|-------------|--------|--------|---------|------|-------|
| **Dev** | $5 (min 0.5 ACU, auto-pause) | $2 | $0.50 | $0.50 | **~$8** |
| **Staging** | $15 (min 1 ACU) | $3 | $1 | $1.50 | **~$21** |
| **Production** | $25 (min 2 ACU, HA) | $5 | $2 | $3 | **~$35** |

> Dùng separate AWS accounts hoặc resource tags cho cost allocation per environment.

---

**Tiếp theo:** [Dọn dẹp tài nguyên](5.17-Cleanup/)