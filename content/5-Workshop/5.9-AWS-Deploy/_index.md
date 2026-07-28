---
title: "AWS Deployment Preparation"
date: 2024-01-01
weight: 9
chapter: false
pre: " <b> 5.9 </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# AWS Deployment Preparation

This section covers building Docker images, pushing to ECR, and preparing Lambda deployment packages for the AWS serverless architecture.

## Prerequisites

- Terraform infrastructure deployed (see [Infrastructure](5.3-Infrastructure/))
- AWS CLI configured with appropriate permissions
- Docker installed and running

## Get Terraform Outputs

```bash
# After terraform apply
ECR_URL=$(terraform output -raw ecr_repository_url)
ECS_CLUSTER=$(terraform output -raw ecs_cluster_name)
RDS_ENDPOINT=$(terraform output -raw rds_endpoint)

echo "ECR: $ECR_URL"
echo "ECS Cluster: $ECS_CLUSTER"
echo "RDS: $RDS_ENDPOINT"
```

## Docker Image Build (Multi-stage)

The `Dockerfile` builds a single image used by all ECS tasks (crawler, etl, vectorize):

```dockerfile
# Dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && \
    rm -rf /var/lib/apt/lists/*

# Copy installed packages
COPY --from=builder /install /usr/local

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Default command (overridden by ECS task definition)
CMD ["python", "main.py", "--mode", "crawl"]
```

## Build and Push to ECR

```bash
# 1. Get ECR login token
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin $ECR_URL

# 2. Build image
docker build -t news-crawler:latest .

# 3. Tag for ECR
docker tag news-crawler:latest $ECR_URL:latest

# 4. Push to ECR
docker push $ECR_URL:latest

# Verify
aws ecr describe-images --repository-name newsrag-api --region ap-southeast-2
```

## Alternative: Buildx for Multi-platform (ARM64 for Fargate Graviton)

```bash
# Create builder
docker buildx create --name multiarch --use

# Build and push for linux/arm64 (Graviton2 - cheaper)
docker buildx build \
  --platform linux/arm64 \
  -t $ECR_URL:latest \
  --push .

# Or build for both
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t $ECR_URL:latest \
  --push .
```

> **Cost Tip:** Graviton2 (ARM64) Fargate is ~20% cheaper than x86. Use `db.t4g.medium` for Aurora and `linux/arm64` for Fargate.

## Lambda Deployment Package

For Lambda functions (Consumer, ETL, RAG API), create deployment packages:

### Directory Structure for Lambda
```
deploy/
├── consumer/
│   ├── lambda_function.py
│   └── requirements.txt
├── etl/
│   ├── lambda_function.py
│   └── requirements.txt
└── rag-api/
    ├── lambda_function.py
    └── requirements.txt
```

### Build Lambda Layer (Shared Dependencies)

```bash
# Create layer with common dependencies
mkdir -p lambda-layer/python
pip install -r requirements.txt -t lambda-layer/python/

# Package layer
cd lambda-layer
zip -r ../lambda-layer.zip python/
cd ..

# Publish layer
aws lambda publish-layer-version \
  --layer-name newsrag-common \
  --zip-file fileb://lambda-layer.zip \
  --compatible-runtimes python3.11 \
  --region ap-southeast-2
```

### Package Individual Lambda Functions

```bash
# Consumer Lambda
cd deploy/consumer
zip -r ../consumer.zip lambda_function.py
cd ../..

# ETL Lambda
cd deploy/etl
zip -r ../etl.zip lambda_function.py
cd ../..

# RAG API Lambda
cd deploy/rag-api
zip -r ../rag-api.zip lambda_function.py
cd ../..
```

## Deploy Lambda Functions (deploy.sh)

The `deploy.sh` script automates Lambda deployment:

```bash
#!/bin/bash
# deploy.sh

set -e

REGION="ap-southeast-2"
RDS_ENDPOINT=$(terraform output -raw rds_endpoint)
DB_PASSWORD=$(terraform output -raw db_password)  # or from Secrets Manager

# Common environment variables
COMMON_ENV="DB_HOST=$RDS_ENDPOINT,DB_NAME=newsrag,DB_USER=postgres,DB_PASSWORD=$DB_PASSWORD,DB_PORT=5432,AWS_REGION=$REGION"

# 1. Consumer Lambda (SQS → Aurora)
aws lambda create-function \
  --function-name newsrag-consumer \
  --runtime python3.11 \
  --handler lambda_function.handler \
  --zip-file fileb://deploy/consumer.zip \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/newsrag-lambda-role \
  --timeout 300 \
  --memory-size 512 \
  --environment Variables="{$COMMON_ENV,SQS_QUEUE_URL=https://sqs.$REGION.amazonaws.com/$(aws sts get-caller-identity --query Account --output text)/newsrag-queue}" \
  --region $REGION

# 2. ETL Lambda (Scheduled → Bedrock Embed → Aurora pgvector)
aws lambda create-function \
  --function-name newsrag-etl \
  --runtime python3.11 \
  --handler lambda_function.handler \
  --zip-file fileb://deploy/etl.zip \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/newsrag-lambda-role \
  --timeout 900 \
  --memory-size 1024 \
  --environment Variables="{$COMMON_ENV,BEDROCK_MODEL_ID=amazon.titan-embed-text-v2:0}" \
  --region $REGION

# 3. RAG API Lambda (API Gateway → Bedrock Embed → pgvector → Groq/Gemini)
aws lambda create-function \
  --function-name newsrag-api \
  --runtime python3.11 \
  --handler lambda_function.handler \
  --zip-file fileb://deploy/rag-api.zip \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/newsrag-lambda-role \
  --timeout 30 \
  --memory-size 512 \
  --environment Variables="{$COMMON_ENV,BEDROCK_MODEL_ID=amazon.titan-embed-text-v2:0,GROQ_API_KEY=$GROQ_API_KEY,GEMINI_API_KEY=$GEMINI_API_KEY}" \
  --region $REGION
```

## IAM Role for Lambda

Create a role with necessary permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "rds-data:ExecuteStatement",
        "rds-data:BatchExecuteStatement",
        "rds-data:BeginTransaction",
        "rds-data:CommitTransaction",
        "rds-data:RollbackTransaction"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:newsrag/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": "arn:aws:bedrock:*:*:foundation-model/amazon.titan-embed-text-v2:0"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:*:*:newsrag-*"
    }
  ]
}
```

## Secrets Manager for Credentials

Store sensitive values in AWS Secrets Manager:

```bash
# Database credentials
aws secretsmanager create-secret \
  --name newsrag/db-credentials \
  --secret-string '{"username":"postgres","password":"your_password","host":"your-aurora-endpoint","port":5432,"dbname":"newsrag"}' \
  --region ap-southeast-2

# API Keys
aws secretsmanager create-secret \
  --name newsrag/api-keys \
  --secret-string '{"groq_api_key":"gsk_...","gemini_api_key":"AIza..."}' \
  --region ap-southeast-2
```

## Verify Deployment

```bash
# Check ECR image
aws ecr describe-images --repository-name newsrag-api --region ap-southeast-2

# Test Lambda Consumer
aws lambda invoke \
  --function-name newsrag-consumer \
  --payload '{"Records":[{"body":"{\"url\":\"https://test.com\",\"title\":\"Test\",\"content\":\"Test content\",\"source_name\":\"Test\",\"source_domain\":\"test.com\",\"category\":\"Test\",\"author\":\"Test\",\"published_at\":\"2024-01-01T00:00:00Z\"}"}]}' \
  /tmp/response.json

# Test ETL Lambda
aws lambda invoke \
  --function-name newsrag-etl \
  --payload '{}' \
  /tmp/etl-response.json

# Check logs
aws logs tail /aws/lambda/newsrag-consumer --follow
aws logs tail /aws/lambda/newsrag-etl --follow
```

## CI/CD with GitHub Actions (Optional)

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: ap-southeast-2

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: newsrag-api
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

      - name: Update ECS Task Definitions
        run: |
          # Update task definition with new image tag
          # Deploy via ECS service update or EventBridge target update
```

## Next Steps

1. **Deploy Infrastructure**: `terraform apply` (see [Infrastructure](5.3-Infrastructure/))
2. **Build & Push Docker**: This section
3. **Deploy Lambdas**: Run `deploy.sh` or use CI/CD
4. **Test Fargate Crawler**: [Fargate Crawler](5.10-Fargate-Crawler/)
5. **Test Lambda Consumer**: [Lambda Consumer](5.11-Lambda-Consumer/)
6. **Test Lambda ETL + Embedding**: [Lambda ETL](5.12-Lambda-ETL/)
7. **Test RAG API**: [RAG API](5.13-RAG-API/)

---

**Next:** [Fargate Crawler](5.10-Fargate-Crawler/)