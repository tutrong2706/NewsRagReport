---
title: "Fargate Crawler (ECS + EventBridge)"
date: 2024-01-01
weight: 10
chapter: false
pre: " <b> 5.10 </b> "
---

# Fargate Crawler: ECS Fargate + EventBridge Scheduler

This section covers deploying the Scrapy crawler as an ECS Fargate task triggered by EventBridge Scheduler.

## Architecture

```
EventBridge Scheduler (01:00 UTC daily)
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

From `main.tf`:

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
# Schedule rule: Daily at 01:00 UTC
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

## IAM Permissions for Task Role

The `ecs_task_role` needs permissions for:
- **SQS**: SendMessage to `news_raw` queue
- **Secrets Manager**: GetSecretValue for DB credentials
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

## Building & Pushing Docker Image

```bash
# 1. Get ECR login
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin $(terraform output -raw ecr_repository_url)

# 2. Build image (from project root)
docker build -t news-crawler .

# 3. Tag for ECR
ECR_URI=$(terraform output -raw ecr_repository_url)
docker tag news-crawler:latest ${ECR_URI}:latest

# 4. Push
docker push ${ECR_URI}:latest

# 5. Verify
aws ecr describe-images --repository-name newsrag-api
```

## Manual Testing

### Run Task Manually

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

### Check Task Status

```bash
# List recent tasks
aws ecs list-tasks --cluster $CLUSTER --family $TASK_DEF --desired-status STOPPED

# Get task details
TASK_ARN=$(aws ecs list-tasks --cluster $CLUSTER --family $TASK_DEF --desired-status STOPPED --query "taskArns[0]" --output text)
aws ecs describe-tasks --cluster $CLUSTER --tasks $TASK_ARN

# View logs
aws logs tail /ecs/newsrag-project --follow --filter-pattern "crawler"
```

## Monitoring & Debugging

### CloudWatch Logs

```bash
# Tail crawler logs
aws logs tail /ecs/newsrag-project --follow --filter-pattern "crawler"

# Search for errors
aws logs filter-log-events \
  --log-group-name /ecs/newsrag-project \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000
```

### CloudWatch Metrics

Create a dashboard with:
- `ECS/TaskCount` (Running, Pending, Stopped)
- `ECS/CPUUtilization` (per task)
- `ECS/MemoryUtilization` (per task)
- `SQS/NumberOfMessagesSent` (news_raw queue)
- `SQS/NumberOfMessagesReceived` (Lambda consumer)

### Common Issues

| Issue | Solution |
|-------|----------|
| `CannotPullContainerError` | Check ECR permissions, image tag exists, VPC has Internet Gateway |
| `Task failed to start` | Check CloudWatch Logs for container error; verify `command` in task definition |
| `SQS AccessDenied` | Attach SQS policy to `ecs_task_role` |
| `Database connection timeout` | Verify security group allows 5432 from ECS SG; check Secrets Manager access |
| `Crawler takes > 15 min` | Increase task CPU/memory; optimize Scrapy settings (`CLOSESPIDER_TIMEOUT`) |

## Scaling Considerations

- **Concurrency**: Run multiple crawler tasks in parallel (different spiders)
- **Queue depth**: Monitor SQS `ApproximateNumberOfMessagesVisible`
- **Cost**: Fargate Spot (up to 70% savings) for fault-tolerant crawling:

```hcl
# In task definition or capacity provider strategy
capacity_provider_strategy {
  capacity_provider = "FARGATE_SPOT"
  weight            = 1
  base              = 0
}
```

## Updating the Crawler

```bash
# 1. Make code changes
# 2. Rebuild & push
docker build -t news-crawler .
docker tag news-crawler:latest ${ECR_URI}:latest
docker push ${ECR_URI}:latest

# 3. Force new deployment (if running as service)
# For scheduled tasks: next EventBridge trigger picks up new image automatically

# 4. Or manually trigger to test
aws ecs run-task --cluster $CLUSTER --task-definition $TASK_DEF ...
```

## Next Steps

After crawler pushes to SQS:
1. **Lambda Consumer** reads SQS → inserts to Aurora: [Lambda Consumer](5.11-Lambda-Consumer/)
2. **Lambda ETL** processes raw → Star Schema + Bedrock Embed: [Lambda ETL](5.12-Lambda-ETL/)
3. **RAG API** serves queries: [RAG API](5.13-RAG-API/)

---

**Next:** [Lambda Consumer (SQS → Aurora)](5.11-Lambda-Consumer/)