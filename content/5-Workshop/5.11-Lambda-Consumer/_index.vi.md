---
title: "Lambda Consumer (SQS → Aurora)"
date: 2024-01-01
weight: 11
chapter: false
pre: " <b> 5.11 </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Thông tin dưới đây chỉ mang tính tham khảo. Vui lòng **không sao chép y nguyên** cho báo cáo của bạn, bao gồm cả cảnh báo này.
{{% /notice %}}

# Lambda Consumer: SQS Trigger → Aurora PostgreSQL

Phần này bao gồm Lambda function consume messages từ SQS và insert bài viết thô vào Aurora PostgreSQL với SHA256 deduplication.

## Kiến trúc

```
SQS Queue (news_raw)
       │
       ▼
┌─────────────────┐
│  Lambda         │  (Python 3.11, 15 min timeout, 512 MB)
│  Consumer       │
│  ─────────────  │
│  • Batch: 10    │
│  • SHA256 dedup │
│  • Insert raw   │
└────────┬────────┘
         │
         ▼
Aurora PostgreSQL (article_metadata table)
```

## Lambda Function Code

### Cấu trúc file
```
deploy/
├── consumer/
│   ├── lambda_function.py
│   └── requirements.txt
```

### lambda_function.py

```python
import hashlib
import json
import logging
import os
import sys
from typing import Dict, Any

import psycopg2
from psycopg2.extras import RealDictCursor

# Add layers path if needed
sys.path.insert(0, "/opt/python")

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def get_db_connection():
    """Tạo database connection dùng environment variables."""
    return psycopg2.connect(
        host=os.environ["DB_HOST"],
        port=os.environ["DB_PORT"],
        database=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
        cursor_factory=RealDictCursor,
    )


def init_database(conn):
    """Đảm bảo article_metadata table tồn tại."""
    with conn.cursor() as cur:
        cur.execute("""
            CREATE TABLE IF NOT EXISTS article_metadata (
                id              BIGSERIAL PRIMARY KEY,
                url_hash        CHAR(64) UNIQUE NOT NULL,
                url             TEXT NOT NULL,
                source_name     VARCHAR(100),
                source_domain   VARCHAR(100),
                title           TEXT,
                content         TEXT,
                category        VARCHAR(100),
                author          VARCHAR(200),
                published_at    TIMESTAMPTZ,
                crawled_at      TIMESTAMPTZ,
                raw_html        TEXT,
                embedded        BOOLEAN DEFAULT FALSE,
                created_at      TIMESTAMPTZ DEFAULT NOW()
            )
        """)
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_article_metadata_url_hash
            ON article_metadata(url_hash)
        """)
        cur.execute("""
            CREATE INDEX IF NOT EXISTS idx_article_metadata_published_at
            ON article_metadata(published_at DESC)
        """)
        conn.commit()


def process_record(record: Dict[str, Any], conn) -> bool:
    """Xử lý một SQS record. Returns True nếu inserted, False nếu duplicate."""
    try:
        # Parse message body
        body = record["body"]
        if isinstance(body, str):
            article = json.loads(body)
        else:
            article = body

        # Validate required fields
        url = article.get("url")
        if not url:
            logger.warning("Record missing URL, skipping")
            return False

        # Compute SHA256 hash cho deduplication
        url_hash = hashlib.sha256(url.encode("utf-8")).hexdigest()

        # Prepare insert data
        insert_data = {
            "url_hash": url_hash,
            "url": url,
            "source_name": article.get("source_name"),
            "source_domain": article.get("source_domain"),
            "title": article.get("title"),
            "content": article.get("content"),
            "category": article.get("category"),
            "author": article.get("author"),
            "published_at": article.get("published_at"),
            "crawled_at": article.get("crawled_at"),
            "raw_html": article.get("raw_html", "")[:50000],  # Limit size
        }

        # Insert với ON CONFLICT DO NOTHING
        with conn.cursor() as cur:
            cur.execute("""
                INSERT INTO article_metadata (
                    url_hash, url, source_name, source_domain, title, content,
                    category, author, published_at, crawled_at, raw_html
                ) VALUES (
                    %(url_hash)s, %(url)s, %(source_name)s, %(source_domain)s,
                    %(title)s, %(content)s, %(category)s, %(author)s,
                    %(published_at)s, %(crawled_at)s, %(raw_html)s
                )
                ON CONFLICT (url_hash) DO NOTHING
                RETURNING id
            """, insert_data)

            result = cur.fetchone()
            conn.commit()

            if result:
                logger.info(f"Inserted article: {url[:80]}...")
                return True
            else:
                logger.debug(f"Duplicate skipped: {url[:80]}...")
                return False

    except json.JSONDecodeError as e:
        logger.error(f"Invalid JSON in record: {e}")
        return False
    except Exception as e:
        logger.exception(f"Error processing record: {e}")
        conn.rollback()
        return False


def lambda_handler(event: Dict[str, Any], context) -> Dict[str, Any]:
    """Lambda entry point - triggered by SQS."""
    logger.info(f"Received {len(event.get('Records', []))} records")

    # Initialize database
    conn = get_db_connection()
    init_database(conn)

    try:
        processed = 0
        inserted = 0
        duplicates = 0
        failed = 0

        for record in event.get("Records", []):
            processed += 1
            if process_record(record, conn):
                inserted += 1
            else:
                duplicates += 1

        logger.info(f"Batch complete: processed={processed}, inserted={inserted}, duplicates={duplicates}")

        return {
            "statusCode": 200,
            "body": json.dumps({
                "processed": processed,
                "inserted": inserted,
                "duplicates": duplicates,
            })
        }

    except Exception as e:
        logger.exception(f"Lambda handler error: {e}")
        return {
            "statusCode": 500,
            "body": json.dumps({"error": str(e)})
        }
    finally:
        conn.close()
```

### requirements.txt

```txt
psycopg2-binary==2.9.9
```

## Terraform Deployment

### Lambda Function Resource

```hcl
resource "aws_lambda_function" "consumer" {
  function_name = "newsrag-consumer"
  description   = "SQS Consumer: inserts raw articles to Aurora"

  runtime       = "python3.11"
  handler       = "lambda_function.lambda_handler"
  timeout       = 900  # 15 minutes
  memory_size   = 512

  # Package as zip
  filename         = "consumer.zip"
  source_code_hash = filebase64sha256("consumer.zip")

  # VPC configuration (same as ECS tasks)
  vpc_config {
    subnet_ids         = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
    security_group_ids = [aws_security_group.lambda_sg.id]
  }

  # IAM role
  role = aws_iam_role.lambda_role.arn

  # Environment variables
  environment {
    variables = {
      DB_HOST     = aws_rds_cluster.main.endpoint
      DB_PORT     = "5432"
      DB_NAME     = "newsrag"
      DB_USER     = var.db_user
      DB_PASSWORD = var.db_password
      # For production, use Secrets Manager ARN instead:
      # DB_SECRET_ARN = aws_secretsmanager_secret.db_credentials.arn
    }
  }
}
```

### SQS Trigger (Event Source Mapping)

```hcl
resource "aws_lambda_event_source_mapping" "consumer_sqs" {
  event_source_arn = aws_sqs_queue.news_raw.arn
  function_name    = aws_lambda_function.consumer.arn
  batch_size       = 10
  maximum_batching_window_in_seconds = 30
  enabled          = true
}
```

### IAM Permissions

```hcl
resource "aws_iam_role_policy" "lambda_permissions" {
  role = aws_iam_role.lambda_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # CloudWatch Logs
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      # VPC ENI management
      {
        Effect = "Allow"
        Action = [
          "ec2:CreateNetworkInterface",
          "ec2:DescribeNetworkInterfaces",
          "ec2:DeleteNetworkInterface",
          "ec2:AssignPrivateIpAddresses",
          "ec2:UnassignPrivateIpAddresses"
        ]
        Resource = "*"
      },
      # SQS (if Lambda needs to send messages elsewhere)
      {
        Effect = "Allow"
        Action = ["sqs:SendMessage"]
        Resource = aws_sqs_queue.news_raw.arn
      },
      # Secrets Manager (for DB credentials)
      {
        Effect = "Allow"
        Action = ["secretsmanager:GetSecretValue"]
        Resource = aws_secretsmanager_secret.db_credentials.arn
      }
    ]
  })
}
```

### Security Group cho Lambda

```hcl
resource "aws_security_group" "lambda_sg" {
  name   = "newsrag-lambda-sg"
  vpc_id = aws_vpc.main.id

  # Allow outbound to Internet (for any external API calls)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Allow Lambda to connect to RDS
resource "aws_security_group_rule" "lambda_to_rds" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.lambda_sg.id
  security_group_id        = aws_security_group.rds_sg.id
}
```

## Building Deployment Package

```bash
# Tạo deployment package
cd deploy/consumer
pip install -r requirements.txt -t . --quiet
zip -r ../consumer.zip lambda_function.py
cd ../..

# Terraform apply sẽ pick up zip mới
terraform apply -auto-approve
```

## deploy.sh Script

```bash
#!/bin/bash
# deploy.sh - Deploy Lambda functions

set -e

# Consumer
echo "Building consumer..."
cd deploy/consumer
pip install -r requirements.txt -t . --quiet
zip -r ../consumer.zip . -q
cd ../..

# ETL (similar)
cd deploy/etl
pip install -r requirements.txt -t . --quiet
zip -r ../etl.zip . -q
cd ../..

# RAG API
cd deploy/rag
pip install -r requirements.txt -t . --quiet
zip -r ../rag.zip . -q
cd ../..

# Terraform apply will pick up new zips
terraform apply -auto-approve
```

## Testing the Consumer

### 1. Send Test Message to SQS

```bash
# Get queue URL
QUEUE_URL=$(aws sqs get-queue-url --queue-name newsrag-news-raw --query "QueueUrl" --output text)

# Send test message
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body '{
    "url": "https://vnexpress.net/test-article-123",
    "source_name": "VnExpress",
    "source_domain": "vnexpress.net",
    "title": "Test Article Title",
    "content": "This is test content for the article.",
    "category": "Test",
    "author": "Test Author",
    "published_at": "2024-01-15T10:30:00+07:00",
    "crawled_at": "2024-01-15T03:30:00Z",
    "raw_html": "<html>...</html>"
  }'
```

### 2. Check Lambda Logs

```bash
# Tail logs
aws logs tail /aws/lambda/newsrag-consumer --follow

# Or filter for recent invocations
aws logs filter-log-events \
  --log-group-name /aws/lambda/newsrag-consumer \
  --start-time $(date -d '10 minutes ago' +%s)000 \
  --filter-pattern "INSERT"
```

### 3. Verify in Database

```bash
ENDPOINT=$(terraform output -raw rds_endpoint)
psql "postgresql://postgres:password@${ENDPOINT}:5432/newsrag" -c "
SELECT source_name, title, embedded, created_at
FROM article_metadata
ORDER BY created_at DESC
LIMIT 10;
"
```

## Monitoring

### CloudWatch Metrics to Watch

| Metric | Namespace | Alarm Threshold |
|--------|-----------|-----------------|
| `Invocations` | AWS/Lambda | - |
| `Errors` | AWS/Lambda | > 0 for 5 min |
| `Duration` (p95) | AWS/Lambda | > 300s |
| `Throttles` | AWS/Lambda | > 0 |
| `ApproximateNumberOfMessagesVisible` | AWS/SQS | > 1000 for 10 min |
| `ApproximateAgeOfOldestMessage` | AWS/SQS | > 300s |

### Alarms

```hcl
resource "aws_cloudwatch_metric_alarm" "consumer_errors" {
  alarm_name          = "newsrag-consumer-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Consumer Lambda errors detected"

  dimensions = {
    FunctionName = aws_lambda_function.consumer.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "sqs_backlog" {
  alarm_name          = "newsrag-sqs-backlog"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Average"
  threshold           = 100
  alarm_description   = "SQS queue backing up - consumer may be down"

  dimensions = {
    QueueName = aws_sqs_queue.news_raw.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `Connection timeout` | Check Lambda SG allows outbound, RDS SG allows inbound from Lambda SG on 5432 |
| `Access denied SecretsManager` | Add `secretsmanager:GetSecretValue` to Lambda role |
| `Duplicate inserts` | Verify `url_hash` UNIQUE constraint and `ON CONFLICT DO NOTHING` |
| `Lambda timeout` | Increase timeout to 900s; reduce batch size; optimize DB inserts |
| `SQS messages not processed` | Check Event Source Mapping is enabled; check Lambda errors |

## Production Hardening

1. **Use RDS Proxy** for connection pooling
2. **Secrets Manager** for DB credentials (not env vars)
3. **Enable X-Ray tracing** for distributed tracing
4. **Set up DLQ** with CloudWatch alarm
5. **Configure retry policy** with exponential backoff

---

**Tiếp theo:** [Lambda ETL + Bedrock Embedding](5.12-Lambda-ETL/)