---
title : "Prerequisites"
date : 2024-01-01 
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

#### Requirements Before Starting

**Accounts and Tools:**
- AWS Account (Free Tier)
- AWS CLI configured (access key, secret key, region `ap-southeast-2`)
- Python 3.10+
- Docker Desktop
- Terraform CLI
- Git

**Clone the repository:**
```bash
git clone <repository-url>
cd NewsRagProject
```

**Set up Python environment:**
```bash
make setup
# or
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
```

**Configure environment variables:**
```bash
cp .env.example .env
# Fill in .env:
# DB_NAME, DB_USER, DB_PASSWORD, DB_HOST, DB_PORT
# QDRANT_HOST, QDRANT_API_KEY
# KAFKA_BOOTSTRAP_SERVERS
# EMBEDDING_MODEL, EMBEDDING_SIZE
# MODEL_1_API_KEY (Groq), MODEL_3_API_KEY (Google)
```

**Start Docker services:**
```bash
docker-compose up -d
# Starts PostgreSQL + Kafka locally
```

**Verify connections:**
```bash
# PostgreSQL
psql -h localhost -U postgres -d newsrag

# Kafka
docker exec -it kafka_newsrag /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```