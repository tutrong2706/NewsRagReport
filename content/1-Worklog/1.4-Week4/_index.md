---
title: "Week 4 Worklog"
date: 2026-07-28
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4 Objectives:

* Package the Crawler into a Docker container.
* Build and push Docker image to Amazon ECR.
* Configure ECS Task Definition for Fargate Crawler.
* Test running the Crawler on Fargate instead of locally.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                              | Start Date   | End Date        | Resources                                 |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Write Dockerfile for Crawler: <br>&emsp; + Base image: `python:3.10-slim` <br>&emsp; + Install system deps: `build-essential`, `libpq-dev` <br>&emsp; + COPY requirements.txt & pip install <br>&emsp; + CMD: `python main.py --mode crawl` | 23/06/2026   | 23/06/2026      | Dockerfile                                |
| Tue | - Build Docker image locally: `docker build -t news-crawler .` <br> - Test container run: `docker run news-crawler` <br> - Debug issues: PYTHONPATH, encoding, env variables          | 24/06/2026   | 24/06/2026      |                                           |
| Wed | - Create ECR repository: `newsrag-api` <br> - Push image to ECR: <br>&emsp; + `aws ecr get-login-password` <br>&emsp; + `docker tag` & `docker push` <br> - Verify image on AWS Console | 25/06/2026   | 25/06/2026      | <https://docs.aws.amazon.com/ecr/>        |
| Thu | - Configure ECS Task Definition for Fargate: <br>&emsp; + CPU: 256 (0.25 vCPU), Memory: 512 MB <br>&emsp; + Network mode: awsvpc <br>&emsp; + Container command: `python main.py --mode crawl` <br>&emsp; + Environment variables from `.env` <br>&emsp; + Log driver: awslogs → CloudWatch | 26/06/2026   | 26/06/2026      | main.tf                                   |
| Fri | - Create ECS Cluster: `newsrag-cluster` <br> - Manually test Fargate Task run <br> - Check logs on CloudWatch <br> - Debug networking issues (Security Groups, VPC, Public IP)        | 27/06/2026   | 27/06/2026      |                                           |


### Week 4 Results:

* Completed optimized Dockerfile:
  ```dockerfile
  FROM python:3.10-slim
  WORKDIR /app
  RUN apt-get update && apt-get install -y build-essential libpq-dev curl
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY . .
  ENV PYTHONPATH=/app
  CMD ["python", "main.py", "--mode", "full"]
  ```

* Docker image builds successfully, runs stably on local

* Successfully pushed image to ECR:
  * Repository: `newsrag-api`
  * Tag: `latest`
  * Size: ~350 MB (after optimization)

* Completed ECS Task Definition configuration:
  * Family: `newsrag-crawler`
  * Fargate compatibility, CPU 256, Memory 512
  * awsvpc network mode for private networking
  * CloudWatch Logs with `crawler` prefix
  * Environment variables injected from Terraform locals

* ECS Cluster `newsrag-cluster` operational, Fargate Task running successfully:
  * Crawler runs inside container on Fargate
  * Logs appearing in CloudWatch
  * Security Group configured to allow egress (page downloads, Kafka communication)
