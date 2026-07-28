---
title: "Week 5 Worklog"
date: 2026-07-28
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5 Objectives:

* Configure Amazon SQS Standard Queue to replace Kafka (cost reduction, simplification).
* Set up Dead Letter Queue (DLQ) for error handling.
* Integrate the Crawler → SQS → Lambda Consumer → PostgreSQL flow.
* End-to-end testing of crawl + consumer flow.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                              | Start Date   | End Date        | Resources                                 |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Create SQS Standard Queue: `newsrag-articles` <br> - Configure message retention: 4 days <br> - Create Dead Letter Queue: `newsrag-articles-dlq` <br> - Configure redrive policy: maxReceiveCount=3 | 14/07/2026   | 14/07/2026      | <https://docs.aws.amazon.com/sqs/>        |
| Tue | - Convert `KafkaPipeline` to `SQSPipeline` (v2): <br>&emsp; + Use `boto3.client('sqs')` <br>&emsp; + `send_message()` instead of Kafka produce <br> - Compare SQS vs Kafka for this use case | 15/07/2026   | 15/07/2026      |                                           |
| Wed | - Study Lambda Consumer (team member B's module): <br>&emsp; + Trigger from SQS event <br>&emsp; + SHA256 hash URL for deduplication <br>&emsp; + Insert into `article_metadata` table <br>&emsp; + ON CONFLICT DO NOTHING | 16/07/2026   | 16/07/2026      | consumer.py                               |
| Thu | - End-to-end integration: Crawler → SQS → Consumer → PostgreSQL <br> - Test flow: crawl 100 articles → check SQS messages → verify database <br> - Debug: message format, encoding, date parsing | 17/07/2026   | 17/07/2026      |                                           |
| Fri | - Study ETL pipeline (team member C's module): <br>&emsp; + `clean_text()`: remove HTML tags, junk patterns <br>&emsp; + `RecursiveCharacterTextSplitter`: chunk 800 chars, overlap 150 <br>&emsp; + Star Schema: `dim_source`, `dim_time`, `dim_author`, `dim_content`, `fact_articles`, `fact_chunks` | 18/07/2026   | 18/07/2026      | etl_warehouse.py, warehouse.sql           |


### Week 5 Results:

* Successfully configured SQS:
  * **SQS Standard Queue**: `newsrag-articles` — cost ~$0/month (vs Kafka requiring management overhead)
  * **Dead Letter Queue**: `newsrag-articles-dlq` — catches failed messages after 3 retries
  * Message retention: 4 days, visibility timeout: 30 seconds

* Clear understanding of Kafka → SQS migration rationale:
  | Criteria        | Kafka (v1)            | SQS (v2)               |
  |-----------------|----------------------|------------------------|
  | Cost            | Container overhead    | ~$0/month              |
  | Management      | Maintain broker       | Fully managed          |
  | Use case        | Real-time streaming   | Daily batch crawl      |
  | Conclusion      | Overkill              | **Best fit**           |

* Successfully integrated end-to-end flow:
  * Crawler crawls articles → serializes JSON → pushes to SQS
  * Lambda Consumer triggered by SQS → parses JSON → SHA256 hashes URL → INSERTs article_metadata
  * Deduplication works: `ON CONFLICT (url_hash) DO NOTHING`
  * Test with 100 articles: 95 successfully inserted, 5 duplicates skipped

* Understood Star Schema warehouse design:
  * **Dimension tables**: `dim_source` (news source), `dim_time` (date), `dim_author` (author), `dim_content` (raw content)
  * **Fact tables**: `fact_articles` (articles), `fact_chunks` (chunks after splitting), `fact_article_authors` (M:N)
  * Indexes: HNSW for vector search, B-tree for url_hash, domain, date
