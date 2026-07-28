---
title: "Clean Up Resources"
date: 2024-01-01
weight: 17
chapter: false
pre: " <b> 5.17 </b> "
---

# Clean Up Resources

This section provides step-by-step instructions to tear down all AWS resources created during the workshop to avoid ongoing charges.

## ⚠️ Before You Begin

**Backup any important data!** This process is **irreversible** and will permanently delete:
- Aurora PostgreSQL database (all articles, chunks, embeddings)
- ECR Docker images
- CloudWatch Logs
- S3 objects (if any)
- Lambda functions and their versions

## Option 1: Terraform Destroy (Recommended)

This destroys all resources managed by Terraform in the correct dependency order.

```bash
# 1. Navigate to project root
cd AWS-Projects

# 2. Review what will be destroyed
terraform plan -destroy

# 3. Destroy (type 'yes' when prompted)
terraform destroy

# 4. Verify all resources are gone
terraform show
```

**Expected output:**
```
Plan: 0 to add, 0 to change, 38 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_lambda_function.rag_api: Destroying... [id=newsrag-rag-api]
aws_lambda_function.etl: Destroying... [id=newsrag-etl]
aws_lambda_function.consumer: Destroying... [id=newsrag-consumer]
aws_ecs_task_definition.crawler: Destroying... [id=newsrag-crawler]
aws_ecs_task_definition.etl: Destroying... [id=newsrag-etl]
aws_cloudwatch_event_target.crawler_target: Destroying... [id=newsrag-crawler-target]
aws_cloudwatch_event_target.etl_target: Destroying... [id=newsrag-etl-target]
aws_cloudwatch_event_rule.crawler_schedule: Destroying... [id=newsrag-crawler-rule]
aws_cloudwatch_event_rule.etl_schedule: Destroying... [id=newsrag-etl-rule]
aws_ecr_repository.api: Destroying... [id=newsrag-api]
aws_ecs_cluster.cluster: Destroying... [id=newsrag-cluster]
aws_db_instance.main: Destroying... [id=newsrag-postgres-1]
aws_rds_cluster.main: Destroying... [id=newsrag-postgres]
...
Destroy complete! Resources: 38 destroyed.
```

## Option 2: Manual Cleanup (If Terraform State Lost)

If you don't have the Terraform state, clean up manually in this order:

### 1. Delete Lambda Functions
```bash
aws lambda delete-function --function-name newsrag-consumer --region ap-southeast-2
aws lambda delete-function --function-name newsrag-etl --region ap-southeast-2
aws lambda delete-function --function-name newsrag-rag-api --region ap-southeast-2
```

### 2. Delete EventBridge Rules & Targets
```bash
# Remove targets first
aws events remove-targets --rule newsrag-crawler-rule --ids crawler-target --region ap-southeast-2
aws events remove-targets --rule newsrag-etl-rule --ids etl-target --region ap-southeast-2
aws events remove-targets --rule newsrag-vectorize-rule --ids vectorize-target --region ap-southeast-2

# Delete rules
aws events delete-rule --name newsrag-crawler-rule --region ap-southeast-2
aws events delete-rule --name newsrag-etl-rule --region ap-southeast-2
aws events delete-rule --name newsrag-vectorize-rule --region ap-southeast-2
```

### 3. Delete ECS Resources
```bash
# Deregister task definitions
aws ecs deregister-task-definition --task-definition newsrag-crawler --region ap-southeast-2
aws ecs deregister-task-definition --task-definition newsrag-etl --region ap-southeast-2
aws ecs deregister-task-definition --task-definition newsrag-vectorize --region ap-southeast-2

# Delete cluster
aws ecs delete-cluster --cluster newsrag-cluster --region ap-southeast-2
```

### 4. Delete ECR Repository
```bash
# Delete all images first
aws ecr batch-delete-image \
  --repository-name newsrag-api \
  --image-ids $(aws ecr list-images --repository-name newsrag-api --query 'imageIds[*]' --output json) \
  --region ap-southeast-2

# Delete repository
aws ecr delete-repository --repository-name newsrag-api --force --region ap-southeast-2
```

### 5. Delete RDS Aurora Cluster
```bash
# Delete instance first
aws rds delete-db-instance \
  --db-instance-identifier newsrag-postgres-1 \
  --skip-final-snapshot \
  --region ap-southeast-2

# Wait for instance deletion, then delete cluster
aws rds delete-db-cluster \
  --db-cluster-identifier newsrag-postgres \
  --skip-final-snapshot \
  --region ap-southeast-2

# Delete subnet group
aws rds delete-db-subnet-group --db-subnet-group-name newsrag-db-subnet --region ap-southeast-2
```

### 6. Delete VPC Resources
```bash
# Get VPC ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=newsrag-vpc" --query "Vpcs[0].VpcId" --output text)

# Delete security groups
aws ec2 delete-security-group --group-id $(aws ec2 describe-security-groups --filters "Name=group-name,Values=newsrag-ecs-sg" --query "SecurityGroups[0].GroupId" --output text) --region ap-southeast-2
aws ec2 delete-security-group --group-id $(aws ec2 describe-security-groups --filters "Name=group-name,Values=newsrag-rds-sg" --query "SecurityGroups[0].GroupId" --output text) --region ap-southeast-2
aws ec2 delete-security-group --group-id $(aws ec2 describe-security-groups --filters "Name=group-name,Values=newsrag-lambda-sg" --query "SecurityGroups[0].GroupId" --output text) --region ap-southeast-2

# Delete subnets
aws ec2 delete-subnet --subnet-id $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=availability-zone,Values=ap-southeast-2a" --query "Subnets[0].SubnetId" --output text) --region ap-southeast-2
aws ec2 delete-subnet --subnet-id $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=availability-zone,Values=ap-southeast-2b" --query "Subnets[0].SubnetId" --output text) --region ap-southeast-2

# Detach and delete IGW
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text)
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID --region ap-southeast-2
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID --region ap-southeast-2

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID --region ap-southeast-2
```

### 7. Delete IAM Roles & Policies
```bash
# Detach policies first
aws iam detach-role-policy --role-name newsrag-ecs-execution-role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy --region ap-southeast-2
aws iam detach-role-policy --role-name newsrag-ecs-task-role --policy-arn arn:aws:iam::123456789012:policy/newsrag-ecs-task-permissions --region ap-southeast-2
aws iam detach-role-policy --role-name newsrag-lambda-role --policy-arn arn:aws:iam::123456789012:policy/newsrag-lambda-permissions --region ap-southeast-2

# Delete inline policies
aws iam delete-role-policy --role-name newsrag-ecs-task-role --policy-name newsrag-ecs-task-permissions --region ap-southeast-2
aws iam delete-role-policy --role-name newsrag-lambda-role --policy-name newsrag-lambda-permissions --region ap-southeast-2

# Delete roles
aws iam delete-role --role-name newsrag-ecs-execution-role --region ap-southeast-2
aws iam delete-role --role-name newsrag-ecs-task-role --region ap-southeast-2
aws iam delete-role --role-name newsrag-lambda-role --region ap-southeast-2
```

### 8. Delete CloudWatch Log Groups
```bash
aws logs delete-log-group --log-group-name /ecs/newsrag-project --region ap-southeast-2
aws logs delete-log-group --log-group-name /aws/lambda/newsrag-consumer --region ap-southeast-2
aws logs delete-log-group --log-group-name /aws/lambda/newsrag-etl --region ap-southeast-2
aws logs delete-log-group --log-group-name /aws/lambda/newsrag-rag-api --region ap-southeast-2
```

### 9. Delete SQS Queues
```bash
aws sqs delete-queue --queue-url https://sqs.ap-southeast-2.amazonaws.com/123456789012/newsrag-news-raw --region ap-southeast-2
aws sqs delete-queue --queue-url https://sqs.ap-southeast-2.amazonaws.com/123456789012/newsrag-news-raw-dlq --region ap-southeast-2
```

### 10. Delete API Gateway
```bash
API_ID=$(aws apigateway get-rest-apis --query "items[?name=='newsrag-api'].id" --output text)
aws apigateway delete-rest-api --rest-api-id $API_ID --region ap-southeast-2
```

### 11. Delete Secrets Manager Secrets
```bash
aws secretsmanager delete-secret --secret-id newsrag/db-credentials --force-delete-without-recovery --region ap-southeast-2
```

### 12. Delete SNS Topic (Alarms)
```bash
aws sns delete-topic --topic-arn arn:aws:sns:ap-southeast-2:123456789012:newsrag-alerts --region ap-southeast-2
```

## Verify Complete Cleanup

```bash
# Check for any remaining resources with "newsrag" tag
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Name,Values=newsrag* \
  --region ap-southeast-2

# Check specific services
aws lambda list-functions --query "Functions[?contains(FunctionName, 'newsrag')].FunctionName" --region ap-southeast-2
aws ecs list-clusters --query "clusterArns[?contains(@, 'newsrag')]" --region ap-southeast-2
aws rds describe-db-clusters --query "DBClusters[?contains(DBClusterIdentifier, 'newsrag')].DBClusterIdentifier" --region ap-southeast-2
aws ecr describe-repositories --query "repositories[?contains(repositoryName, 'newsrag')].repositoryName" --region ap-southeast-2
aws sqs list-queues --queue-name-prefix newsrag --region ap-southeast-2
aws apigateway get-rest-apis --query "items[?contains(name, 'newsrag')].id" --region ap-southeast-2
```

All should return empty results `[]`.

## Clean Up Local Files

```bash
# Remove Terraform state (if not using remote backend)
rm -rf .terraform terraform.tfstate terraform.tfstate.backup

# Remove deployment packages
rm -f consumer.zip etl.zip rag.zip

# Remove Docker images
docker rmi news-crawler:latest
docker rmi $(docker images -q -f dangling=true)

# Remove Python virtual environment
rm -rf venv

# Remove .env files (optional, keep for next time)
# rm -f .env terraform.tfvars
```

## Cost Verification

After cleanup, verify no ongoing charges:

1. **AWS Billing Dashboard** → Check "Cost Explorer" for current month
2. **Set up Budget Alert** for $1/month to catch any stray resources:
   ```bash
   aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) --budget file://budget.json
   ```

## Common Issues

| Issue | Solution |
|-------|----------|
| `DependencyViolation` deleting VPC | Delete ENIs first: `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=$VPC_ID` then `aws ec2 delete-network-interface` |
| `DBInstanceNotFound` | Instance already deleted, skip to cluster deletion |
| `ResourceInUseException` (Lambda) | Wait 1-2 minutes for EventBridge targets to detach |
| `InvalidParameterException` (RDS) | Ensure `skip-final-snapshot` is used for dev environments |

---

## Workshop Complete! 🎉

You have successfully built and deployed a **production-ready News RAG Pipeline on AWS** with:

- ✅ **Infrastructure as Code** (Terraform)
- ✅ **Serverless Architecture** (Fargate, Lambda, Aurora Serverless v2)
- ✅ **Event-Driven Pipeline** (EventBridge → SQS → Lambda → Bedrock → pgvector)
- ✅ **RAG API** (Vector search + LLM generation)
- ✅ **Monitoring & Alerting** (CloudWatch, Logs, Metrics)
- ✅ **Cost Optimization** (~$21-26/month)
- ✅ **CI/CD Ready** (GitHub Actions)

### Next Steps for Production

1. **Add Authentication** - API Gateway Authorizers (Cognito/JWT)
2. **Custom Domain** - Route 53 + ACM certificate for API Gateway
3. **WAF** - Protect API from abuse
4. **Multi-AZ Aurora** - Enable `storage_encrypted`, `deletion_protection`
5. **RDS Proxy** - For connection pooling at scale
6. **Secrets Rotation** - Automatic credential rotation
7. **Blue/Green Deploy** - ECS deployment circuits
8. **Chaos Engineering** - Test failure scenarios

---

**Thank you for completing the workshop!**

*Repository: https://github.com/your-org/AWS-Projects*