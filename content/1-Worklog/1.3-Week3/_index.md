---
title: "Week 3 Worklog"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives:

* Develop the complete `NewsRAGSpider` — the main spider for news crawling.
* Implement article information extraction logic: title, content, author, publish_date.
* Write `KafkaPipeline` to push crawled data into Kafka topic.
* Test crawling locally.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                              | Start Date   | End Date        | Resources                                 |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Write `NewsRAGSpider` class inheriting `scrapy.Spider` <br> - Implement `parse()` method: filter internal URLs, classify article links (.html, .htm) and category links <br> - Configure `custom_settings`: CONCURRENT_REQUESTS, DOWNLOAD_DELAY, DEPTH_LIMIT | 30/06/2025   | 30/06/2025      | spider.py                                 |
| Tue | - Implement `parse_article()`: use `newspaper3k` to parse HTML <br> - Filter articles with content < 100 characters <br> - Extract author from `article.authors`                     | 01/07/2025   | 01/07/2025      |                                           |
| Wed | - Advanced author extraction: <br>&emsp; + Fallback through multiple CSS selectors (`.author-name`, `.tac-gia`, `a[rel="author"]`,...) <br>&emsp; + Validate author: filter out fake authors (URLs, dates, newspaper names) <br>&emsp; + `is_valid_author()` function: check length, special characters, bad words | 02/07/2025   | 02/07/2025      |                                           |
| Thu | - Publish_date parsing: <br>&emsp; + Parse ISO format: `2025-07-01T14:30` <br>&emsp; + Parse VN format: `01/07/2025 14:30` <br>&emsp; + Fallback through multiple CSS selectors + `article.publish_date` <br> - Write `KafkaPipeline`: serialize item to JSON → push to Kafka topic `news_raw` | 03/07/2025   | 03/07/2025      | pipelines.py                              |
| Fri | - Test local crawl: `scrapy crawl news_rag_spider` <br> - Verify output: correct JSON format, valid author, proper dates <br> - Debug and fix CSS selectors for each newspaper       | 04/07/2025   | 04/07/2025      |                                           |


### Week 3 Results:

* Completed `NewsRAGSpider` (`crawler/spiders/spider.py`) with features:
  * **parse()**: Traverses all links on page, filters same-domain URLs, classifies articles vs category pages
  * **parse_article()**: Uses `newspaper3k` combined with CSS Selectors
  * **Custom settings**: CONCURRENT_REQUESTS=16 (Windows) / 32 (Linux), DOWNLOAD_DELAY=1.0/0.5, DEPTH_LIMIT=5

* Built complex author extraction system with multi-level fallback:
  1. Extract from `newspaper3k` → validate
  2. Fallback through 12+ CSS selectors: `.author-name`, `.tac-gia`, `a[href*="tac-gia"]`,...
  3. Scan bottom paragraphs for patterns like "Theo: ..." / "Nguon: ..."
  4. Final fallback to `meta[name="author"]`
  * `is_valid_author()` checks: length 2-100 chars, no URLs/emails, no date patterns, no bad words (newspaper names, generic labels)

* Handled diverse date parsing:
  * ISO format: `2025-07-01T14:30:00`
  * VN format: `01/07/2025 14:30` or `01-07-2025`
  * Fallback through 10 CSS selectors + `article.publish_date`
  * Standardized output: `YYYY-MM-DD HH:MM:SS`

* Completed `KafkaPipeline` (`crawler/pipelines.py`):
  * Serializes item to JSON (ensure_ascii=False for Vietnamese)
  * Pushes to Kafka topic `news_raw`
  * Producer flush after each item

* Successfully tested local crawling with valid output format
