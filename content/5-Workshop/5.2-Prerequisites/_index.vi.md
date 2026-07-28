---
title: "Điều kiện tiên quyết"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2 </b> "
---

{{% notice warning %}}
⚠️ **Lưu ý:** Thông tin dưới đây chỉ mang tính tham khảo. Vui lòng **không sao chép y nguyên** cho báo cáo của bạn, bao gồm cả cảnh báo này.
{{% /notice %}}

# Điều kiện tiên quyết

Trước khi bắt đầu workshop này, hãy đảm bảo bạn đã cài đặt và cấu hình các công cụ sau.

## Tài khoản & Quyền cần thiết

### Tài khoản AWS
- **Tài khoản AWS** với quyền quản trị hoặc quyền cho các dịch vụ:
  - VPC, EC2, Subnets, Route Tables, Internet Gateway
  - RDS (Aurora PostgreSQL Serverless v2)
  - ECS (Fargate, Clusters, Task Definitions, Services)
  - ECR (Repositories)
  - Lambda (Functions, Layers, Permissions)
  - SQS (Queues)
  - EventBridge (Rules, Schedules, Targets)
  - API Gateway (REST APIs)
  - Bedrock (Truy cập model: `amazon.titan-embed-text-v2:0`)
  - IAM (Roles, Policies)
  - CloudWatch (Log Groups, Metrics)
  - CloudFormation (Stacks)

> **Mẹo:** Sử dụng IAM user có `AdministratorAccess` cho workshop, hoặc đảm bảo role của bạn có các quyền trên.

### API Keys bên ngoài (cho RAG)
- **Groq API Key** — Lấy từ [console.groq.com](https://console.groq.com/) (Có gói miễn phí)
- **Google Gemini API Key** — Lấy từ [aistudio.google.com](https://aistudio.google.com/) (Có gói miễn phí)

## Công cụ bắt buộc

### 1. AWS CLI v2
```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# Kiểm tra
aws --version
# Nên ra: aws-cli/2.x.x
```

**Cấu hình AWS CLI:**
```bash
aws configure
# Nhập: AWS Access Key ID, Secret Access Key, Region (ví dụ: ap-southeast-2), Output format (json)
```

### 2. Terraform >= 1.5.0
```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Windows (Chocolatey)
choco install terraform

# Kiểm tra
terraform version
# Nên ra: Terraform v1.5.x hoặc cao hơn
```

### 3. Docker & Docker Compose
```bash
# macOS (Docker Desktop)
brew install --cask docker

# Linux
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Đăng xuất và đăng nhập lại

# Docker Compose (thường có sẵn với Docker Desktop)
# Linux độc lập:
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Kiểm tra
docker --version
docker compose version
```

### 4. Python 3.10+
```bash
# macOS
brew install python@3.11

# Linux (Ubuntu/Debian)
sudo apt update && sudo apt install python3.11 python3.11-venv python3.11-dev

# Windows
# Tải từ python.org

# Kiểm tra
python3 --version
# Nên là 3.10.x hoặc cao hơn
```

### 5. Git
```bash
# macOS
brew install git

# Linux
sudo apt install git

# Windows
# Tải từ git-scm.com

# Kiểm tra
git --version
```

### 6. Trình soạn thảo mã (Khuyến nghị: VS Code)
```bash
# macOS
brew install --cask visual-studio-code

# Linux
# Tải từ code.visualstudio.com

# Extensions khuyến nghị:
# - HashiCorp Terraform
# - Docker
# - Python
# - AWS Toolkit
# - YAML
```

## Thiết lập dự án

### Clone Repository
```bash
git clone <your-repo-url> AWS-Projects
cd AWS-Projects
```

### Môi trường ảo Python
```bash
python3 -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows

pip install --upgrade pip
pip install -r requirements.txt
```

### Biến môi trường
Sao chép file mẫu và điền giá trị của bạn:
```bash
cp .env.example .env
```

Chỉnh sửa `.env` với thông tin xác thực:
```env
# Database (Aurora - điền sau khi Terraform apply)
DB_NAME=newsrag
DB_USER=postgres
DB_PASSWORD=your_secure_password
DB_HOST=your-aurora-endpoint.cluster-xyz.ap-southeast-2.rds.amazonaws.com
DB_PORT=5432

# Qdrant (chỉ cho dev local)
QDRANT_HOST=localhost
QDRANT_PORT=6333
QDRANT_COLLECTION_NAME=news_chunks
QDRANT_API_KEY=

# Kafka (chỉ cho dev local)
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_TOPIC_NEWS=news_raw

# Embedding Model (dev local)
EMBEDDING_MODEL=BAAI/bge-small-en-v1.5
EMBEDDING_SIZE=384

# LLM APIs (Lấy từ respective consoles)
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxx
GEMINI_API_KEY=AIza_xxxxxxxxxxxxxxxxxxxx

# AWS
AWS_REGION=ap-southeast-2
```

### Biến Terraform
Tạo `terraform.tfvars`:
```bash
cat > terraform.tfvars << EOF
db_password     = "your_secure_password_123!"
db_user         = "postgres"
qdrant_api_key  = ""  # Để trống cho triển khai AWS
qdrant_host     = ""  # Để trống cho triển khai AWS
model_1_api_key = "gsk_xxxxxxxxxxxxxxxxxxxx"  # Groq
model_2_api_key = "gsk_xxxxxxxxxxxxxxxxxxxx"  # Groq backup
model_3_api_key = "AIza_xxxxxxxxxxxxxxxxxxxx" # Gemini
EOF
```

> **Bảo mật:** Không bao giờ commit `.env` hoặc `terraform.tfvars` lên git. Chúng đã có trong `.gitignore`.

## Xác minh thiết lập

Chạy script kiểm tra:
```bash
make verify
```

Hoặc kiểm tra thủ công từng công cụ:
```bash
aws sts get-caller-identity  # Nên hiển thị AWS account của bạn
terraform version            # Nên >= 1.5.0
docker run hello-world       # Nên in "Hello from Docker!"
python3 -c "import scrapy; print('Scrapy OK')"
```

## Khu vực AWS (Region)

Workshop này sử dụng **ap-southeast-2 (Sydney)** mặc định. Để thay đổi:
1. Cập nhật `provider "aws" { region = "your-region" }` trong `main.tf`
2. Cập nhật `AWS_REGION` trong `.env`
3. Đảm bảo model Bedrock `amazon.titan-embed-text-v2:0` có sẵn trong region của bạn (kiểm tra [Bedrock regions](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html))

## Truy cập Model Bedrock

Bật quyền truy cập model trong AWS Console:
1. Vào **Amazon Bedrock** → **Model access**
2. Nhấn **Manage model access**
3. Bật **Amazon Titan Embeddings G1 - Text v2** (`amazon.titan-embed-text-v2:0`)
4. Đợi trạng thái hiển thị "Access granted"

---

**Tiếp theo:** [Hạ tầng dưới dạng Code (Terraform)](5.3-Infrastructure/)