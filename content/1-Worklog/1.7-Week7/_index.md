---
title: "Week 7 Worklog"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7 Objectives:

* Run integration tests across the full pipeline: Crawl → ETL → Vectorize → RAG API.
* Debug and handle edge cases during crawling (author, date, encoding).
* Study the Ragas evaluation framework and compare NewsRAG with FlashRAG.
* Optimize crawling performance and error handling.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                              | Start Date   | End Date        | Resources                                 |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Integration test: run full pipeline `python main.py --mode full` <br> - Verify flow: Spider → Kafka → Consumer → PostgreSQL → ETL → Vectorize <br> - Monitor CloudWatch logs      | 28/07/2025   | 28/07/2025      |                                           |
| Tue | - Debug edge cases in spider: <br>&emsp; + Author containing URLs, dates, special chars <br>&emsp; + Articles with no title or content < 100 chars <br>&emsp; + Non-standard date formats from different newspapers <br>&emsp; + UTF-8 encoding issues with Vietnamese | 29/07/2025   | 29/07/2025      |                                           |
| Wed | - Improve author validation in spider: <br>&emsp; + Add `fake_authors` list (vietnamnet news, ban bien tap,...) <br>&emsp; + Enhance `is_valid_author()`: filter names with bad_words (weekdays, dates,...) <br>&emsp; + Add bottom paragraph scanning for "Theo: ..." pattern | 30/07/2025   | 30/07/2025      |                                           |
| Thu | - Study Ragas evaluation framework: <br>&emsp; + Faithfulness: truthfulness against context <br>&emsp; + Answer Relevancy: relevance to question <br>&emsp; + Context Precision: document ranking <br>&emsp; + Context Recall: document coverage <br> - Compare NewsRAG vs FlashRAG results | 31/07/2025   | 31/07/2025      | Pipeline_v3.md                            |
| Fri | - Optimize Crawler: <br>&emsp; + Adjust CONCURRENT_REQUESTS, DOWNLOAD_DELAY per OS <br>&emsp; + Improve retry handling on request failure <br>&emsp; + Add ROBOTSTXT_OBEY config <br> - Re-test full pipeline after optimization | 01/08/2025   | 01/08/2025      |                                           |


### Week 7 Results:

* Successful integration test on full pipeline:
  * Crawls ~500 articles per run from 3 newspapers
  * ETL processes: clean HTML → chunk 800 chars (overlap 150) → insert Star Schema
  * Vectorize: embedding with `BAAI/bge-small-en-v1.5` → upsert to Qdrant
  * RAG API: query → embed → search top-k → LLM generates answer

* Handled critical edge cases:
  * **Invalid author**: Detected and removed 15+ types of fake authors (URLs, dates, newspaper names, special chars)
  * **Date parsing**: Supports 3 formats (ISO, VN dd/mm/yyyy, date-only) + fallback through 10 CSS selectors
  * **Empty content**: Skips articles with content < 100 characters
  * **ETL author cleanup**: Removes authors > 40 chars, containing http/|/@

* Ragas evaluation results:
  | Metric                | NewsRAG        | FlashRAG       |
  |----------------------|----------------|----------------|
  | **Context Precision** | **Higher**     | Average        |
  | **Context Recall**    | Average        | **Higher**     |
  | **Faithfulness**      | Average        | **Higher**     |
  | **Answer Relevancy**  | Comparable     | Comparable     |

  > Overall metrics are modest (below 0.5) because the RAG prompt is configured to provide detailed, comprehensive answers → reduces cosine similarity when Ragas reverse-translates to questions.

* Crawler performance optimization:
  * Windows: CONCURRENT_REQUESTS=16, DOWNLOAD_DELAY=1.0
  * Linux (Fargate): CONCURRENT_REQUESTS=32, DOWNLOAD_DELAY=0.5
  * OS-specific User-Agent customization
