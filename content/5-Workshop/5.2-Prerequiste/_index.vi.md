---
title : "Chuẩn bị"
date : 2024-01-01 
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

#### Yêu cầu trước khi bắt đầu

**Tài khoản và công cụ:**
- AWS Account (Free Tier)
- AWS CLI đã cấu hình (access key, secret key, region `ap-southeast-2`)
- Python 3.10+
- Docker Desktop
- Terraform CLI
- Git

**Clone repository:**
```bash
git clone <repository-url>
cd NewsRagProject
```

**Thiết lập môi trường Python:**
```bash
make setup
# hoặc
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
```

**Cấu hình biến môi trường:**
```bash
cp .env.example .env
# Điền thông tin vào .env:
# DB_NAME, DB_USER, DB_PASSWORD, DB_HOST, DB_PORT
# QDRANT_HOST, QDRANT_API_KEY
# KAFKA_BOOTSTRAP_SERVERS
# EMBEDDING_MODEL, EMBEDDING_SIZE
# MODEL_1_API_KEY (Groq), MODEL_3_API_KEY (Google)
```

**Khởi chạy Docker services:**
```bash
docker-compose up -d
# Khởi tạo PostgreSQL + Kafka local
```

**Kiểm tra kết nối:**
```bash
# PostgreSQL
psql -h localhost -U postgres -d newsrag

# Kafka
docker exec -it kafka_newsrag /opt/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```