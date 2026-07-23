---
title : "Deploy"
date : 2024-01-01 
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

#### Deploying Infrastructure as Code with Terraform

Instead of manually configuring each service on the AWS Console, the NewsRAG project uses Terraform to automate the provisioning of the entire infrastructure (Infrastructure as Code - IaC).

**1. Terraform Structure**

The `main.tf` file (over 370 lines) manages all resources:
- **Networking**: VPC (`10.0.0.0/16`), 2 Public Subnets (multi-AZ), Internet Gateway, Route Tables.
- **Security Groups**: Access control following the Principle of Least Privilege — e.g., RDS only accepts traffic from the ECS Security Group.
- **Database**: RDS Aurora PostgreSQL Serverless cluster.
- **Compute**: ECS Cluster, ECR Repository, and Task Definitions for Crawler, ETL, Vectorize running on Fargate.
- **Scheduling**: EventBridge Rules to trigger ECS Tasks via cron schedules (01:00, 02:00, 03:00 UTC).

**2. Deploying to AWS**

To deploy the system, you only need to configure the AWS CLI and run these commands:

```bash
# Initialize Terraform provider
terraform init

# Preview the resources to be created
terraform plan

# Execute resource creation on AWS
terraform apply -auto-approve
```

**3. Deployment Automation (CI/CD)**

The `deploy.sh` script is provided to automatically build Docker images, tag and push them to Amazon ECR, and then update the ECS service to force a deployment of the latest version:

```bash
#!/bin/bash
# deploy.sh

# 1. Authenticate with ECR
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com

# 2. Build Docker images
docker build -t newsrag-crawler -f crawler/Dockerfile .
docker build -t newsrag-etl -f etl/Dockerfile .

# 3. Tag and Push to ECR
docker tag newsrag-crawler:latest $ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/newsrag-api:crawler-latest
docker push $ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/newsrag-api:crawler-latest
```

**4. Resource Cleanup**

When the system is not in use, to avoid incurring costs, you can delete all resources with a single command:

```bash
terraform destroy
```

> **Note**: This command will delete all VPC, ECS, RDS, and EventBridge resources defined in `main.tf`. Database data will be lost if you haven't configured a snapshot.