---
title: "Blog 1: Overcoming Timeout with AWS Fargate"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Overcoming AWS Lambda's 15-minute Limit with ECS Fargate Architecture

During the development of the **News RAG Pipeline**, one of the biggest technical challenges we faced was automating the crawling of news from major Vietnamese websites (VnExpress, Thanh Nien, VietnamNet). Initially, the system was designed to run on **AWS Lambda**.

### The Problem: Execution Timeout

AWS Lambda is a fantastic service for running serverless code. However, it has a **hard limit of 15 minutes (900 seconds)** per execution.

When our crawler parses sitemaps of news websites, the number of articles to process can reach thousands daily. Fetching HTML content, parsing data, and sending it to a queue takes a significant amount of time, especially since we must respect the origin server's rate limits to avoid being blocked. As a result, the Lambda function frequently **timed out** before completing its job.

### The Solution: Migrating to Amazon ECS Fargate

To solve this problem while maintaining the "Serverless" philosophy (avoiding 24/7 running EC2 instances), we decided to migrate our crawling architecture to **Amazon ECS Fargate**.

**Why Fargate?**
1. **No Execution Time Limits**: Fargate tasks can run as long as needed until the crawling job is finished.
2. **Scheduled Execution**: By integrating with **Amazon EventBridge**, we easily scheduled the Crawler to run automatically at 01:00 and 02:00 AM UTC every day.
3. **Easy Scaling**: If we need to crawl more websites, we can simply adjust the CPU and RAM allocation for the Task (currently, 0.25 vCPU and 0.5 GB RAM are sufficient).
4. **Docker Packaging**: The Scrapy framework and its dependencies are neatly packaged in a Docker Image and stored in **Amazon ECR**.

### Results Achieved

Migrating to ECS Fargate brought absolute stability to our data ingestion pipeline. The crawler can now run continuously for 30-40 minutes every night, fetching hundreds of new articles without any interruptions. Furthermore, the cost for Fargate is highly optimized since we only pay for the exact compute minutes the Crawler actually runs.

*Key takeaway: No single AWS service is a "silver bullet." Understanding the limits of services like Lambda and flexibly switching to more appropriate services like ECS Fargate is a crucial skill in Cloud architecture design.*

![News RAG Pipeline Architecture](/images/5-Workshop/5.1-Workshop-overview/blog1.jpeg)