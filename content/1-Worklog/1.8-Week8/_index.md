---
title: "Week 8 Worklog"
date: 2026-07-28
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Week 8 Objectives:

* Finalize code, clean up and optimize.
* Detailed comparison of architecture v1 vs v2 (cost reduction ~30%).
* Write technical documentation `Pipeline_v3.md`.
* Complete the internship report.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                              | Start Date   | End Date        | Resources                                 |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Code review and clean up: <br>&emsp; + Remove dead code, unnecessary comments <br>&emsp; + Standardize coding style (PEP 8) <br>&emsp; + Verify `.env.example` has all variables  | 04/08/2025   | 04/08/2025      |                                           |
| Tue | - Detailed v1 vs v2 comparison: <br>&emsp; + Crawler: Lambda (15 min timeout) → Fargate (unlimited) <br>&emsp; + Stream: Kafka → SQS (~$0) <br>&emsp; + Embedding: bge-m3 local → Bedrock API <br>&emsp; + Vector DB: Qdrant Cloud → Aurora pgvector <br>&emsp; + Cost: ~$35/month → ~$21-26/month (reduced ~30%) | 05/08/2025   | 05/08/2025      | Pipeline_v3.md                            |
| Wed | - Write technical documentation `Pipeline_v3.md`: <br>&emsp; + Overall architecture (text diagram) <br>&emsp; + v1 vs v2 comparison table <br>&emsp; + Component map <br>&emsp; + Technical details for each module <br>&emsp; + Operational costs <br>&emsp; + Ragas evaluation results | 06/08/2025   | 06/08/2025      |                                           |
| Thu | - Write team task distribution & timeline in Pipeline_v3.md <br> - Write future development directions <br> - Write deployment and local development guide                           | 07/08/2025   | 07/08/2025      |                                           |
| Fri | - Complete internship report (Hugo report) <br> - Review all 8 weeks of worklog content <br> - Prepare presentation slides (if needed) <br> - Submit report                          | 08/08/2025   | 08/08/2025      |                                           |


### Week 8 Results:

* Finalized production-quality code:
  * All modules have error handling, logging
  * `.env.example` with all 50 environment variables
  * `Makefile` with 12 convenient commands (setup, up, down, full, auto, crawl, etl,...)
  * `deploy.sh` for automated deployment

* Architecture v1 vs v2 comparison:

  | Component       | v1 (Thesis)                       | v2 (Improved)                     | Reason                                |
  |----------------|----------------------------------|----------------------------------|---------------------------------------|
  | Crawler         | Lambda + CrawlSpider (15 min)    | **Fargate + SitemapSpider**      | No timeout, crawls old articles       |
  | Stream          | Kafka on Docker                  | **SQS Standard (~$0)**           | Overkill for batch pipeline           |
  | Embedding       | BAAI/bge-m3 (1024d, local)       | **Bedrock Titan Embed v2**       | Serverless, no model loading          |
  | Vector DB       | Qdrant Cloud (3rd party)         | **Aurora pgvector**              | Leverage RDS, no external dependency  |
  | **Total Cost**  | **~$35/month**                   | **~$21-26/month**                | **Reduced ~30%**                      |

* Completed `Pipeline_v3.md` documentation (460 lines) including:
  * Overall architecture with text diagram
  * 5 technical module details with code samples
  * SQL Schema (Star Schema + pgvector)
  * Detailed operational costs
  * Ragas evaluation results and FlashRAG comparison
  * Future development directions

* Completed internship report:
  * 8 weeks of detailed worklog
  * NewsRAG project proposal
  * Hands-on workshop
  * Self-evaluation and feedback
