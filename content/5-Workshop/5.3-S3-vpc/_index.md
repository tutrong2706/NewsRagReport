---
title : "Crawler Module"
date : 2024-01-01 
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

#### Building Automated Crawler with Scrapy and ECS Fargate

The Crawler module is responsible for collecting new articles daily and pushing them to the SQS queue for processing. Using ECS Fargate eliminates the 15-minute timeout limit of AWS Lambda (v1 architecture).

**1. Scrapy SitemapSpider**

Unlike CrawlSpider which relies on links from the homepage, SitemapSpider directly reads the `sitemap_news.xml` file of newspapers to ensure no older articles are missed.

```python
# crawler/spiders/spider.py (Excerpt)
import scrapy
from newspaper import Article
from crawler.utils import is_valid_author

class NewsRAGSpider(scrapy.Spider):
    name = 'news_rag_spider'
    
    def parse(self, response):
        # Filter internal URLs and classify articles
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

**2. Docker Container Packaging**

```dockerfile
# crawler/Dockerfile
FROM python:3.10-slim
WORKDIR /app

# Install system dependencies for psycopg2
RUN apt-get update && apt-get install -y build-essential libpq-dev

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
ENV PYTHONPATH=/app
CMD ["python", "main.py", "--mode", "crawl"]
```

**3. Amazon SQS Integration**

Crawled data is serialized to JSON and pushed to SQS instead of Kafka to save costs:

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