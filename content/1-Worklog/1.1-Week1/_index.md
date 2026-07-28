---
title: "Week 1 Worklog"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Week 1 Objectives:

* Get familiar with the First Cloud AI Journey (FCAJ) program and team members.
* Learn basic AWS services related to the project.
* Understand the overall architecture of NewsRAG project (original thesis version — v1).
* Clone the repository and set up the local development environment.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                                       | Start Date   | End Date        | Resources                                 |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Meet FCAJ team members <br> - Read program regulations and rules <br> - Receive assignments and project team distribution                                                                  | 16/06/2025   | 16/06/2025      |                                           |
| Tue | - Study AWS overview and service categories: <br>&emsp; + Compute (EC2, ECS, Fargate, Lambda) <br>&emsp; + Storage (S3) <br>&emsp; + Database (RDS, Aurora) <br>&emsp; + Networking (VPC, SG) <br>&emsp; + Messaging (SQS, Kafka) | 17/06/2025   | 17/06/2025      | <https://cloudjourney.awsstudygroup.com/> |
| Wed | - Create AWS Free Tier account <br> - Install AWS CLI & configure credentials <br> - Explore AWS Management Console                                                                          | 18/06/2025   | 18/06/2025      | <https://cloudjourney.awsstudygroup.com/> |
| Thu | - Read NewsRAG v1 architecture documentation (original thesis) <br> - Study RAG (Retrieval-Augmented Generation) architecture <br> - Research components: Crawler, Kafka, ETL, Vector DB, LLM | 19/06/2025   | 19/06/2025      | Pipeline_v3.md                            |
| Fri | - Clone NewsRAG repository <br> - Set up Python virtual environment <br> - Run `docker-compose up -d` to initialize PostgreSQL + Kafka locally <br> - Read and understand source code structure | 20/06/2025   | 20/06/2025      | README.md, Makefile                       |


### Week 1 Results:

* Understood AWS overview and service categories relevant to the NewsRAG project:
  * **Compute**: EC2, ECS Fargate (container execution), Lambda (serverless functions)
  * **Database**: RDS Aurora PostgreSQL (article storage + vector search)
  * **Messaging**: SQS (message queue), Kafka (streaming — used in v1)
  * **Container**: ECR (registry), ECS (orchestration)

* Successfully created and configured AWS Free Tier account, including:
  * Access Key / Secret Key
  * Default region: `ap-southeast-2`
  * AWS CLI working on terminal

* Understood the overall NewsRAG architecture:
  * **v1 (Thesis)**: Lambda Crawler + Kafka + BAAI/bge-m3 + Qdrant Cloud + FastAPI/Next.js
  * **v2 (Improved)**: Fargate Crawler + SQS + Bedrock Titan Embed + Aurora pgvector
  * Module distribution: Crawl+Queue (A), Consumer+DB (B), ETL+Embed (C), RAG+Frontend (D)

* Successfully cloned the repository and set up local environment:
  * Python 3.10 virtual environment
  * PostgreSQL + Kafka running via Docker Compose
  * Understood directory structure: `crawler/`, `consumer/`, `etl/`, `vectorize/`, `search/`, `database/`
