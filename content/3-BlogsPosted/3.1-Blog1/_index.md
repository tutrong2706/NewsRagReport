---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Building an Automated News Crawling System with Scrapy on AWS ECS Fargate

In the NewsRAG project, the team needed an automated, stable news crawling system running daily to collect data from 3 major Vietnamese newspapers: **VnExpress**, **Thanh Niên**, **VietnamNet**. This blog shares the experience of building a crawler system using the Scrapy framework, packaging it in Docker, and deploying on AWS ECS Fargate.

### Why Scrapy?

- **Powerful and flexible**: Supports Spider, Pipeline, Middleware — easy to extend
- **Concurrent processing**: CONCURRENT_REQUESTS adjustable (16–32 simultaneous requests)
- **Rich ecosystem**: SitemapSpider, CrawlSpider, many extensions
- **Great compatibility with newspaper3k**: Vietnamese content extraction library

### Challenges in Crawling Vietnamese News

1. **Author extraction**: Each newspaper has different HTML structure. Built a fallback system through 12+ CSS selectors
2. **Date parsing**: Support both ISO format and VN format (dd/mm/yyyy HH:MM)
3. **Fake authors**: Many cases where author is the newspaper name or generic label → need `is_valid_author()` filter
4. **Encoding**: Ensure UTF-8 for Vietnamese throughout the entire pipeline

### Crawler Architecture

```
Scrapy Spider → KafkaPipeline → Kafka/SQS → Consumer → PostgreSQL
     ↓
parse_article()
     ↓
newspaper3k + CSS Selectors
     ↓
{title, content, author, publish_date, url, source}
```

### Deployment on ECS Fargate

- **Dockerfile**: Base `python:3.10-slim`, install `libpq-dev` for psycopg2
- **ECS Task Definition**: 0.25 vCPU, 512 MB RAM (sufficient for crawling)
- **EventBridge**: Auto-runs at 01:00 UTC daily
- **CloudWatch Logs**: Real-time monitoring

### Results

- Crawls ~500 articles per run from 3 sources
- Runtime: ~30 minutes
- Fargate cost: ~$3-5/month

...Images...

...Blog link...