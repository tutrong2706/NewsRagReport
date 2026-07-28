---
title: "Fargate Crawler (ECS + EventBridge)"
date: 2026-07-28
weight: 10
chapter: false
pre: " <b> 5.10 </b> "
---

# Fargate Crawler: ECS Fargate + EventBridge Scheduler

Phần này bao gồm việc deploy Scrapy crawler như ECS Fargate task được trigger bởi EventBridge Scheduler.

## Kiến trúc

```
EventBridge Scheduler (01:00 UTC hàng ngày)
       │
       ▼
ECS RunTask API
       │
       ▼
┌─────────────────┐
│  Fargate Task   │  (newsrag-crawler task definition)
│  ─────────────  │
│  Python main.py │
│  --mode crawl   │
└─────────────────┘
       │
       ▼
SQS Queue (news_raw)
       │
       ▼
Lambda Consumer → Aurora PostgreSQL
```

## Task Definition (Terraform)

Từ `main.tf`:

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

## EventBridge Scheduler (Terraform)

```hcl
# Schedule rule: Hàng ngày lúc 01:00 UTC
resource "aws_cloudwatch_event_rule" "crawler_schedule" {
  name                = "newsrag-crawler-rule"
  schedule_expression = "cron(0 1 * * ? *)"
}

# Target: ECS RunTask
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
```

## IAM Permissions cho Task Role

`ecs_task_role` cần permissions cho:
- **SQS**: SendMessage tới `news_raw` queue
- **Secrets Manager**: GetSecretValue cho DB credentials
- **CloudWatch Logs**: CreateLogStream, PutLogEvents
- **ECR**: BatchGetImage, GetDownloadUrlForLayer (via execution role)

```hcl
resource "aws_iam_role_policy" "ecs_task_permissions" {
  name = "newsrag-ecs-task-permissions"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "sqs:SendMessage",
          "sqs:GetQueueAttributes"
        ]
        Resource = aws_sqs_queue.news_raw.arn
      },
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = aws_secretsmanager_secret.db_credentials.arn
      }
    ]
  })
}
```

## SQS Queue (Terraform)

```hcl
resource "aws_sqs_queue" "news_raw" {
  name                       = "newsrag-news-raw"
  visibility_timeout_seconds = 300
  message_retention_seconds  = 1209600  # 14 days
  fifo_queue                 = false

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.news_raw_dlq.arn
    maxReceiveCount     = 3
  })
}

resource "aws_sqs_queue" "news_raw_dlq" {
  name = "newsrag-news-raw-dlq"
}
```

## Build & Push Docker Image

```bash
# 1. Lấy ECR login
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin $(terraform output -raw ecr_repository_url)

# 2. Build image (từ project root)
docker build -t news-crawler .

# 3. Tag cho ECR
ECR_URI=$(terraform output -raw ecr_repository_url)
docker tag news-crawler:latest ${ECR_URI}:latest

# 4. Push
docker push ${ECR_URI}:latest

# 5. Verify
aws ecr describe-images --repository-name newsrag-api --region ap-southeast-2
```

## Alternative: Buildx cho Multi-platform (ARM64 cho Graviton)

```bash
# Tạo builder
docker buildx create --name multiarch --use

# Build và push cho linux/arm64 (Graviton2 - rẻ hơn)
docker buildx build \
  --platform linux/arm64 \
  -t ${ECR_URI}:latest \
  --push .

# Hoặc build cho cả hai
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ${ECR_URI}:latest \
  --push .
```

> **Cost Tip:** Graviton2 (ARM64) Fargate ~20% rẻ hơn x86. Dùng `db.t4g.medium` cho Aurora và `linux/arm64` cho Fargate.

## Manual Testing

### Chạy Task Thủ Công

```bash
CLUSTER=$(terraform output -raw ecs_cluster_name)
TASK_DEF=newsrag-crawler
SUBNET_A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=newsrag-pub-a" --query "Subnets[0].SubnetId" --output text)
SUBNET_B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=newsrag-pub-b" --query "Subnets[0].SubnetId" --output text)
SG=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=newsrag-ecs-sg" --query "SecurityGroups[0].GroupId" --output text)

aws ecs run-task \
  --cluster $CLUSTER \
  --task-definition $TASK_DEF \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_A,$SUBNET_B],securityGroups=[$SG],assignPublicIp=ENABLED}" \
  --overrides '{"containerOverrides":[{"name":"crawler","command":["python","main.py","--mode","crawl"]}]}'
```

### Kiểm Tra Trạng Thái Task

```bash
# Liệt kê tasks gần đây
aws ecs list-tasks --cluster $CLUSTER --family $TASK_DEF --desired-status STOPPED

# Chi tiết task
TASK_ARN=$(aws ecs list-tasks --cluster $CLUSTER --family $TASK_DEF --desired-status STOPPED --query "taskArns[0]" --output text)
aws ecs describe-tasks --cluster $CLUSTER --tasks $TASK_ARN

# Xem logs
aws logs tail /ecs/newsrag-project --follow --filter-pattern "crawler"
```

## Monitoring & Debugging

### CloudWatch Logs

```bash
# Tail crawler logs
aws logs tail /ecs/newsrag-project --follow --filter-pattern "crawler"

# Tìm lỗi
aws logs filter-log-events \
  --log-group-name /ecs/newsrag-project \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000
```

### CloudWatch Metrics

Tạo dashboard với:
- `ECS/TaskCount` (Running, Pending, Stopped)
- `ECS/CPUUtilization` (per task)
- `ECS/MemoryUtilization` (per task)
- `SQS/NumberOfMessagesSent` (news_raw queue)
- `SQS/NumberOfMessagesReceived` (Lambda consumer)

### Vấn đề thường gặp

| Vấn đề | Giải pháp |
|--------|-----------|
| `CannotPullContainerError` | Check ECR permissions, image tag exists, VPC có Internet Gateway |
| `Task failed to start` | Check CloudWatch Logs cho container error; verify `command` trong task definition |
| `SQS AccessDenied` | Attach SQS policy vào `ecs_task_role` |
| `Database connection timeout` | Verify security group cho phép 5432 từ ECS SG; check Secrets Manager access |
| `Crawler takes > 15 min` | Tăng task CPU/memory; optimize Scrapy settings (`CLOSESPIDER_TIMEOUT`) |

## Scaling Considerations

- **Concurrency:** Chạy multiple crawler tasks song song (khác spiders)
- **Queue depth:** Monitor SQS `ApproximateNumberOfMessagesVisible`
- **Cost:** Fargate Spot (lên đến 70% tiết kiệm) cho fault-tolerant crawling:

```hcl
# Trong task definition hoặc capacity provider strategy
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
```

## Cập nhật Crawler

```bash
# 1. Sửa code
# 2. Rebuild & push
docker build -t news-crawler .
docker tag news-crawler:latest ${ECR_URI}:latest
docker push ${ECR_URI}:latest

# 3. Force new deployment (nếu chạy as service)
# Cho scheduled tasks: EventBridge trigger tiếp theo sẽ pick up image mới tự động

# 4. Hoặc trigger thủ công để test
aws ecs run-task --cluster $CLUSTER --task-definition $TASK_DEF ...
```

## Bước tiếp theo

Sau khi crawler đẩy lên SQS:
1. **Lambda Consumer** đọc SQS → insert Aurora: [Lambda Consumer](5.11-Lambda-Consumer/)
2. **Lambda ETL** xử lý raw → Star Schema + Bedrock Embed: [Lambda ETL](5.12-Lambda-ETL/)
3. **RAG API** serve queries: [RAG API](5.13-RAG-API/)

---

**Tiếp theo:** [Lambda Consumer (SQS → Aurora)](5.11-Lambda-Consumer/)