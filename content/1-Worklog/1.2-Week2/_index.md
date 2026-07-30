---
title: "Week 2 Worklog"
date: 2026-07-28
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives:

* Deep-dive into Scrapy framework — the main tool for news crawling.
* Understand the differences between CrawlSpider (v1) and SitemapSpider (v2).
* Master Docker, Dockerfile, docker-compose for containerization preparation.
* Analyze sitemap XML structure of 3 Vietnamese news outlets.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                            | Start Date   | End Date        | Resources                                 |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Study Scrapy framework: <br>&emsp; + Scrapy project structure <br>&emsp; + Spider, Pipeline, Settings <br>&emsp; + Middleware and Item Pipeline                                 | 09/06/2026   | 09/06/2026      | <https://docs.scrapy.org/>                |
| Tue | - Compare CrawlSpider vs SitemapSpider: <br>&emsp; + CrawlSpider: relies on DEPTH_LIMIT, misses old articles <br>&emsp; + SitemapSpider: reads all URLs from sitemap, navigation-independent | 10/06/2026   | 10/06/2026      | <https://docs.scrapy.org/>                |
| Wed | - Analyze sitemap XML structure of 3 news outlets: <br>&emsp; + VnExpress: `sitemap_news.xml` <br>&emsp; + Thanh Niên: `sitemap.xml` <br>&emsp; + VietnamNet: `sitemap_news.xml`   | 11/06/2026   | 11/06/2026      |                                           |
| Thu | - Learn Docker basics: <br>&emsp; + Dockerfile: FROM, WORKDIR, COPY, RUN, CMD <br>&emsp; + docker-compose.yml: services, volumes, environment <br>&emsp; + Multi-stage build      | 12/06/2026   | 12/06/2026      | <https://docs.docker.com/>                |
| Fri | - Study `newspaper3k` library for article content extraction <br> - Research Kafka Producer/Consumer pattern <br> - Learn `confluent_kafka` Python client                          | 13/06/2026   | 13/06/2026      | <https://newspaper.readthedocs.io/>       |


### Week 2 Results:

* Mastered Scrapy project structure:
  * `spiders/`: contains spider classes
  * `pipelines.py`: processes items after spider yields
  * `settings.py`: configures CONCURRENT_REQUESTS, DOWNLOAD_DELAY, DEPTH_LIMIT,...

* Understood the key differences between 2 spider types:
  * **CrawlSpider (v1)**: Uses `DEPTH_LIMIT=5`, only crawls pages within link scope → misses old articles not linked from homepage
  * **SitemapSpider (v2)**: Reads directly from sitemap XML file → crawls all articles including old ones, avoids Lambda's 15-minute timeout

* Analyzed sitemap XML of 3 news outlets:
  * VnExpress: `https://vnexpress.net/sitemap_news.xml`
  * Thanh Niên: `https://thanhnien.vn/sitemap.xml`
  * VietnamNet: `https://vietnamnet.vn/sitemap_news.xml`

* Acquired necessary Docker knowledge:
  * How to write optimized Dockerfile (base image `python:3.10-slim`, multi-layer caching)
  * Docker Compose to run PostgreSQL + Kafka locally
  * Understanding `.env` files and how to inject into containers

* Studied `newspaper3k`:
  * Python library for extracting title, content, author, publish_date from HTML
  * Uses `article.set_html()` + `article.parse()` instead of direct download
  * Combined with Scrapy's CSS Selectors for improved accuracy
