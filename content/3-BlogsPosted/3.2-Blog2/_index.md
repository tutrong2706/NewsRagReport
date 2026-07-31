---
title: "Blog 2: Cost Optimization with SQS"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# System Cost Optimization: Migrating from Apache Kafka to Amazon SQS

A crucial criterion when designing systems on the Cloud is **Cost Optimization**. In the first version (v1) developed in our local environment, the project used **Apache Kafka** as the message broker to stream data from the Crawler to the Database. However, when migrating to AWS, cost became a significant barrier.

### The Cost Challenge with Kafka

Apache Kafka is an incredibly powerful stream processing tool. But to run Kafka on AWS, we generally have two main choices:
1. **Amazon MSK (Managed Streaming for Apache Kafka)**: Very expensive (can cost dozens or hundreds of USD per month).
2. **Self-hosting Kafka on EC2**: Requires maintaining EC2 instances running 24/7, incurring fixed costs (at least ~$10-15/month) plus the overhead of system administration, security updates, and managing ZooKeeper.

Meanwhile, the data flow of NewsRAG is fundamentally **Batch processing**: The Crawler only runs 1-2 times a day (at night) and pushes around 500 - 1000 new articles. Maintaining a Kafka cluster running 24/7 just to serve this intermittent data flow is extremely wasteful.

### The Alternative: Amazon SQS (Simple Queue Service)

We decided to completely replace the queue architecture with **Amazon SQS**.

**Why SQS is the perfect fit:**
1. **Fully Serverless & Pay-as-you-go**: SQS requires no servers. You only pay based on the number of requests. With our system's daily news volume, the number of requests easily fits within the **Free Tier** (the first 1 million requests per month are free). The queue cost was reduced to **essentially $0**.
2. **Native Integration with AWS Lambda**: SQS can act as an "Event Source" to automatically trigger a Lambda function (Lambda Consumer). When the Crawler pushes an article to SQS, SQS automatically invokes Lambda to save it into the Database, without us having to write polling loops.
3. **Dead-Letter Queue (DLQ)**: SQS provides a built-in DLQ mechanism. If an article is malformed and Lambda fails to save it to the DB after several retries, that message is moved to the DLQ so we can analyze it later, ensuring no data loss.

### Design Evaluation

By removing Kafka and switching to SQS, the system became not only lighter and easier to maintain but also saved a massive amount in operational budget. This is a practical proof that: **Choosing technology is not about picking the "fanciest" tool, but choosing the most "appropriate" one for the current scale and problem.**

![News RAG Pipeline Architecture](/images/5-Workshop/5.1-Workshop-overview/blog2.jpeg)