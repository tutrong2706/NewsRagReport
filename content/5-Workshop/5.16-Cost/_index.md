---
title: "Cost Optimization"
date: 2024-01-01
weight: 16
chapter: false
pre: " <b> 5.16 </b> "
---

# Cost Optimization

This section covers strategies to minimize AWS costs for the News RAG Pipeline while maintaining performance.

## Cost Breakdown (Monthly, ap-southeast-2)

| Service | Configuration | Est. Cost | % of Total |
|---------|--------------|-----------|------------|
| **Aurora Serverless v2** | 2 ACU baseline, auto-scale to 8 | $15-20 | ~50% |
| **ECS Fargate (Crawler)** | 0.25 vCPU, 0.5 GB, 30 min/day | $0.50 | ~1% |
| **ECS Fargate (ETL)** | 0.5 vCPU, 1 GB, 30 min/day | $1.00 | ~3% |
| **Lambda (Consumer)** | 512 MB, 1M req, 5s avg | $0.30 | ~1% |
| **Lambda (ETL)** | 1 GB, 30/day, 300s avg | $1.50 | ~4% |
| **Lambda (RAG API)** | 1 GB, 10K req, 3s avg | $0.50 | ~1% |
| **API Gateway** | 10K req/month | $0.35 | ~1% |
| **SQS** | 30K messages/month | $0.00 | ~0% |
| **Bedrock Titan Embed** | 500K tokens/day | $0.75 | ~2% |
| **ECR** | 2 GB storage | $0.20 | ~0% |
| **CloudWatch Logs** | 1 GB ingest, 7-day retention | $1.50 | ~4% |
| **Data Transfer** | < 1 GB/month | $0.09 | ~0% |
| **SNS (Alarms)** | < 100 notifications | $0.01 | ~0% |
| **Total** | | **~$21-26/month** | **100%** |

> **Comparison:** Previous v1 architecture (Kafka, Qdrant Cloud, local embeddings) cost ~$35/month. **Savings: ~30-40%**

## Optimization Strategies

### 1. Aurora Serverless v2

```hcl
# Terraform: Set capacity range
resource "aws_rds_cluster" "main" {
  # ...
  serverlessv2_scaling_configuration {
    min_capacity = 0.5   # Minimum ACUs (can scale to 0 with pause)
    max_capacity = 8     # Maximum ACUs
  }
}

# Enable auto-pause for dev environments
resource "aws_rds_cluster" "main" {
  # ...
  # For production, don't auto-pause (causes cold start)
  # For dev/staging:
  # engine_mode = "provisioned"  # or use provisioned for predictable workloads
}
```

**Tips:**
- Use **ACU = 0.5** minimum for dev (scales to 0 when idle)
- Production: **min=1, max=16** for HA
- Monitor `ServerlessDatabaseCapacity` metric to tune
- Consider **Provisioned** with `db.t4g.medium` (~$45/mo) if steady high load

### 2. Lambda Right-Sizing

```bash
# Use AWS Lambda Power Tuning to find optimal memory
# https://github.com/aws-lambda-power-tuning/aws-lambda-power-tuning

# Typical results:
# - Consumer: 512 MB (CPU scales with memory)
# - ETL: 1024 MB (needs memory for pgvector ops + Bedrock)
# - RAG API: 1024 MB (embedding + search + LLM)
```

**Cost vs Performance:**
| Memory | CPU | ETL Duration | Cost/Invocation |
|--------|-----|--------------|-----------------|
| 512 MB | 0.8 vCPU | 180s | $0.0000018 |
| 1024 MB | 1.7 vCPU | 90s | $0.0000018 |
| 2048 MB | 3.5 vCPU | 50s | $0.0000020 |

→ **1024 MB is sweet spot** for ETL (2x faster, same cost)

### 3. Fargate Spot for Crawler

```hcl
# Add capacity provider strategy for Spot
resource "aws_ecs_service" "crawler" {
  # ... (if running as service)
  # For scheduled tasks, use capacity provider in RunTask override
}

# In EventBridge target:
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

**Savings:** Up to 70% on Fargate compute (~$0.15/month for crawler)

### 4. Bedrock Embedding Optimization

```python
# Batch embeddings where possible (but Bedrock invoke_model is single)
# Optimize: reduce chunk size, filter before embedding

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
        if len(chunk.strip()) > 30:
            chunks.append(chunk)
    return chunks
```

**Savings:** ~$0.25/month (30% fewer embeddings)

### 5. CloudWatch Logs Retention

```hcl
resource "aws_cloudwatch_log_group" "logs" {
  name              = "/ecs/newsrag-project"
  retention_in_days = 3  # Reduced from 7 for dev; 14 for prod
}
```

**Savings:** 50% log cost reduction ($0.75 vs $1.50/month)

### 6. S3 for Cold Data (Optional)

```hcl
# Archive old chunks to S3 Glacier
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

Move `fact_chunks` older than 30 days to S3, keep only recent in pgvector.

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

**When to use:** > 50 concurrent Lambda connections
**Cost:** ~$0.015/hr per vCPU = ~$11/month for db.t4g.medium equivalent
**Benefit:** Connection pooling, reduced cold start, better scalability

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
# Daily cost by service (last 30 days)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Monthly forecast
aws ce get-cost-forecast \
  --time-period Start=$(date +%Y-%m-01),End=$(date -d '+1 month' +%Y-%m-01) \
  --metric BLENDED_COST \
  --granularity MONTHLY
```

## Cost Optimization Checklist

| ✅ | Optimization | Status | Savings |
|---|-------------|--------|---------|
| 1 | Aurora min ACU = 0.5 (dev) |  | $10/mo |
| 2 | Lambda memory right-sized |  | $0.50/mo |
| 3 | Fargate Spot for crawler |  | $0.35/mo |
| 4 | Chunk size 300 tokens |  | $0.25/mo |
| 5 | CloudWatch logs 3-day retention (dev) |  | $0.75/mo |
| 6 | Budget alert at 80% |  | Prevention |
| 7 | RDS Proxy (if >50 concurrent) |  | Scalability |
| 8 | S3 Glacier for old data |  | Future |

**Total Potential Savings:** ~$12-15/month (50% reduction) for dev environment

## Production vs Development Costs

| Environment | Aurora | Lambda | Fargate | Logs | Total |
|-------------|--------|--------|---------|------|-------|
| **Dev** | $5 (min 0.5 ACU, auto-pause) | $2 | $0.50 | $0.50 | **~$8** |
| **Staging** | $15 (min 1 ACU) | $3 | $1 | $1.50 | **~$21** |
| **Production** | $25 (min 2 ACU, HA) | $5 | $2 | $3 | **~$35** |

> Use separate AWS accounts or resource tags for cost allocation per environment.

---

**Next:** [Clean Up Resources](5.17-Cleanup/)