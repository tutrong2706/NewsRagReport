---
title: "Blog 3"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Deploying Infrastructure as Code with Terraform for AWS Projects

In the NewsRAG project, the entire AWS infrastructure is managed using **Terraform** — the most popular Infrastructure as Code (IaC) tool. This blog shares how to organize Terraform config for a real project with VPC, RDS, ECS, and EventBridge.

### Why Infrastructure as Code?

- **Reproducible**: Recreate entire infrastructure with a single `terraform apply`
- **Version control**: Track changes through Git
- **Automation**: No manual clicking on AWS Console
- **Team collaboration**: Review infrastructure changes via Pull Request

### Terraform Structure for NewsRAG

```
main.tf (371 lines)
├── Provider: aws (ap-southeast-2)
├── Variables: db_password, qdrant_api_key, model_api_keys (sensitive)
├── VPC & Networking
│   ├── VPC (10.0.0.0/16)
│   ├── 2 Public Subnets (2a, 2b)
│   ├── Internet Gateway + Route Table
│   └── Security Groups (ECS, RDS)
├── RDS Aurora PostgreSQL
│   ├── Cluster: aurora-postgresql 15.4
│   └── Instance: db.t4g.medium
├── ECS + ECR
│   ├── ECR Repository
│   ├── ECS Cluster
│   ├── 3 Task Definitions (crawler, etl, vectorize)
│   └── IAM Roles
├── EventBridge Scheduler
│   ├── Crawler: 01:00 UTC
│   ├── ETL: 02:00 UTC
│   └── Vectorize: 03:00 UTC
└── Outputs: rds_endpoint, ecr_url, cluster_name
```

### Best Practices Applied

1. **Sensitive variables**: Use `type = string, sensitive = true` for passwords and API keys
2. **Locals block**: Manage shared environment variables via `locals.common_env`
3. **Security Groups**: Principle of least privilege — RDS only accepts from ECS SG
4. **CloudWatch Logs**: 7-day retention to save costs
5. **Outputs**: Export RDS endpoint, ECR URL for CI/CD usage

### Results

- Single `main.tf` managing 20+ AWS resources
- Deploy/destroy infrastructure in < 10 minutes
- Easy to recreate new environments

...Images...

...Blog link...