---
title: "Week 6 Worklog"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives:

* Write Terraform `main.tf` to provision all AWS infrastructure.
* Configure EventBridge Scheduler for automated pipeline execution.
* Set up IAM Roles, Security Groups, VPC for ECS + RDS.
* Test terraform apply and verify resources on AWS Console.

### Tasks for this week:
| Day | Tasks                                                                                                                                                                              | Start Date   | End Date        | Resources                                 |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| Mon | - Write Terraform — VPC & Networking: <br>&emsp; + VPC `newsrag-vpc` (CIDR 10.0.0.0/16) <br>&emsp; + 2 Public Subnets (ap-southeast-2a, 2b) <br>&emsp; + Internet Gateway + Route Table <br>&emsp; + Security Groups: `ecs_sg` (egress all), `rds_sg` (ingress 5432 from ECS) | 21/07/2025   | 21/07/2025      | main.tf                                   |
| Tue | - Write Terraform — RDS Aurora PostgreSQL: <br>&emsp; + Cluster: `newsrag-postgres`, engine aurora-postgresql 15.4 <br>&emsp; + Instance: `db.t4g.medium` (2 vCPU, 4GB RAM) <br>&emsp; + DB Subnet Group for multi-AZ | 22/07/2025   | 22/07/2025      | main.tf                                   |
| Wed | - Write Terraform — ECS + ECR: <br>&emsp; + ECR Repository: `newsrag-api` <br>&emsp; + ECS Cluster: `newsrag-cluster` <br>&emsp; + 3 Task Definitions: crawler (256 CPU/512 MB), etl (512/1024), vectorize (512/1024) <br>&emsp; + IAM Roles: execution role + task role <br>&emsp; + CloudWatch Log Group: `/ecs/newsrag-project` | 23/07/2025   | 23/07/2025      | main.tf                                   |
| Thu | - Write Terraform — EventBridge Scheduler: <br>&emsp; + Crawler: `cron(0 1 * * ? *)` — daily at 01:00 UTC <br>&emsp; + ETL: `cron(0 2 * * ? *)` — 02:00 UTC <br>&emsp; + Vectorize: `cron(0 3 * * ? *)` — 03:00 UTC <br>&emsp; + Each rule targets the corresponding ECS Fargate Task | 24/07/2025   | 24/07/2025      | main.tf                                   |
| Fri | - `terraform init` & `terraform plan` — review changes <br> - `terraform apply` — deploy infrastructure <br> - Verify resources on AWS Console <br> - Test manual EventBridge trigger | 25/07/2025   | 25/07/2025      |                                           |


### Week 6 Results:

* Completed `main.tf` (371 lines) managing all infrastructure:
  * **VPC**: 1 VPC, 2 public subnets, Internet Gateway, Route Table
  * **Security Groups**: ECS (egress all), RDS (ingress 5432 only from ECS SG)
  * **RDS**: Aurora PostgreSQL 15.4, instance `db.t4g.medium`, skip final snapshot
  * **ECR**: Repository `newsrag-api`
  * **ECS**: Cluster + 3 Task Definitions (crawler, etl, vectorize)
  * **IAM**: Execution role (ECS Task Execution Policy) + Task role
  * **CloudWatch**: Log group `/ecs/newsrag-project`, retention 7 days
  * **EventBridge**: 3 scheduled rules

* EventBridge Scheduler configuration:
  | Rule Name                | Cron Expression       | Time (UTC) | Target Task        |
  |--------------------------|----------------------|------------|-------------------|
  | `newsrag-crawler-rule`   | `cron(0 1 * * ? *)`  | 01:00      | newsrag-crawler   |
  | `newsrag-etl-rule`       | `cron(0 2 * * ? *)`  | 02:00      | newsrag-etl       |
  | `newsrag-vectorize-rule` | `cron(0 3 * * ? *)`  | 03:00      | newsrag-vectorize |

* Environment variables managed via `locals.common_env`:
  * DB credentials → automatically uses RDS endpoint
  * Qdrant, Kafka, Embedding, LLM configs
  * Sensitive values (passwords, API keys) → Terraform variables (type=string, sensitive=true)

* Outputs: `rds_endpoint`, `ecr_repository_url`, `ecs_cluster_name`

* `terraform apply` succeeded, all resources created on AWS
