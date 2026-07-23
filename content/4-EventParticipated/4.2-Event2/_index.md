---
title: "Event 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 4.2. </b> "
---

# Summary Report: “AWS Serverless & Container Day”

### Event Objectives

- Introduce the latest trends in Serverless and Container technologies on AWS.
- Deep dive into Amazon ECS, EKS, and AWS Fargate.
- Guide on building microservices architectures optimized for cost and performance.
- Share practical experiences from AWS experts and customers.

### Key Highlights

#### 1. Container Optimization with AWS Fargate
- **No server management**: Fargate allows developers to focus on the application rather than worrying about managing EC2 instances.
- **Right-sizing feature**: Configuring precise CPU and Memory for each task to save costs.
- **Graviton Integration**: Running Fargate on ARM-based processors (Graviton) boosts performance by 40% at a 20% lower cost.

#### 2. Event-Driven Architecture with EventBridge
- Using Amazon EventBridge to schedule or respond to events from other services (S3, SQS).
- How EventBridge reduces the need for integration code between microservices.

#### 3. AWS Lambda Best Practices
- **Cold Start** Management: Using Provisioned Concurrency.
- Choosing the right memory size to achieve the perfect balance between performance and cost.
- Timeout limits: Keeping in mind Lambda's 15-minute limit for long-running tasks.

### Applying to Work (NewsRAG Project)

- **Crawler Architecture Transition**: The knowledge about Lambda's 15-minute limit made me realize that NewsRAG v1's architecture (using Lambda for web crawling) was unviable for SitemapSpider. From this event, I proposed and directly transitioned the Crawler to run on containers using **AWS Fargate**, completely resolving the timeout issue.
- **Pipeline Automation**: Applied **Amazon EventBridge Scheduler** to automatically trigger the Fargate Crawler daily at 01:00 UTC instead of running it manually.
- **Docker Cost Optimization**: Based on best practices from the event, I optimized the Dockerfile for the Crawler (using `python:3.10-slim`) and set Fargate configuration to the smallest footprint (0.25 vCPU, 512MB RAM) since the I/O bound task didn't require much compute.

### Event Experience

The **"AWS Serverless & Container Day"** event took place right when the NewsRAG team was stuck with Lambda Crawler timeout errors. The knowledge gained acted as a literal lifesaver.

#### Interaction & Q&A
- I had the opportunity to ask AWS Solutions Architects directly about long-running web scraping problems and received clear advice: *"Use ECS Fargate instead of Lambda for tasks with unpredictable completion times"*.

#### Technical Networking
- Met many other developers and heard their stories about container configuration mistakes, helping me avoid similar pitfalls when writing the `main.tf` Terraform file for my project.

#### Core Takeaway
- Every technology has its specific use case. Lambda is fantastic for APIs (like our RAG API), but Fargate is the true solution for long-running batch jobs like Crawler and ETL.

#### Event photos
*Add your real event photos here*

> The event was a major turning point in designing the NewsRAG v2 system, bringing stability and superior performance to the entire data pipeline.
