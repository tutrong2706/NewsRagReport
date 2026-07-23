---
title : "Module Crawler"
date : 2024-01-01 
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

#### Xây dựng Crawler tự động với Scrapy và ECS Fargate

Module Crawler chịu trách nhiệm thu thập tin tức mới mỗi ngày và đẩy vào hàng đợi SQS để xử lý. Việc sử dụng ECS Fargate giúp loại bỏ giới hạn timeout 15 phút của AWS Lambda (kiến trúc v1).

**1. Scrapy SitemapSpider**

Khác với CrawlSpider phụ thuộc vào các link trên trang chủ, SitemapSpider đọc trực tiếp file `sitemap_news.xml` của các báo để đảm bảo không bỏ sót bài cũ.

```python
# crawler/spiders/spider.py (Trích đoạn)
import scrapy
from newspaper import Article
from crawler.utils import is_valid_author

class NewsRAGSpider(scrapy.Spider):
    name = 'news_rag_spider'
    
    def parse(self, response):
        # Lọc URL nội bộ và phân biệt bài viết
        ...
        
    def parse_article(self, response):
        article = Article(response.url)
        article.set_html(response.text)
        article.parse()
        
        # Multi-fallback author extraction
        author = article.authors[0] if article.authors else None
        if not author or not is_valid_author(author):
            author = response.css('.author-name::text').get()
            
        yield {
            'title': article.title,
            'content': article.text,
            'author': author,
            'publish_date': article.publish_date,
            'url': response.url,
            'source': self.allowed_domains[0]
        }
```

**2. Đóng gói Docker Container**

```dockerfile
# crawler/Dockerfile
FROM python:3.10-slim
WORKDIR /app

# Cài đặt system dependencies cho psycopg2
RUN apt-get update && apt-get install -y build-essential libpq-dev

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
ENV PYTHONPATH=/app
CMD ["python", "main.py", "--mode", "crawl"]
```

**3. Tích hợp Amazon SQS**

Dữ liệu crawl được sẽ serialize thành JSON và đẩy vào SQS thay vì Kafka để tiết kiệm chi phí:

```python
# crawler/pipelines.py
import boto3
import json

class SQSPipeline:
    def __init__(self):
        self.sqs = boto3.client('sqs', region_name='ap-southeast-2')
        self.queue_url = 'https://sqs.ap-southeast-2.amazonaws.com/123456789/newsrag-articles'
        
    def process_item(self, item, spider):
        self.sqs.send_message(
            QueueUrl=self.queue_url,
            MessageBody=json.dumps(dict(item), ensure_ascii=False)
        )
        return item
```