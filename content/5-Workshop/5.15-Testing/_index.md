---
title: "Testing & Monitoring"
date: 2024-01-01
weight: 15
chapter: false
pre: " <b> 5.15 </b> "
---

{{% notice warning %}}
⚠️ **Note:** The information below is for reference purposes only. Please **do not copy verbatim** for your report, including this warning.
{{% /notice %}}

# Testing & Monitoring

This section covers end-to-end testing, RAG evaluation with RAGAS, and production monitoring setup.

## Test Suite Overview

| Test Type | Tool | Purpose |
|-----------|------|---------|
| Unit Tests | pytest | Individual function correctness |
| Integration Tests | pytest + Docker | Pipeline stages locally |
| E2E Tests | Playwright | Full user flows |
| RAG Evaluation | RAGAS | Faithfulness, Relevancy, Precision, Recall |
| Load Testing | Locust | API performance under load |
| Chaos Testing | AWS FIS | Resilience validation |

## 1. Unit & Integration Tests (Local)

### pytest Configuration

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short --strict-markers
markers =
    unit: Unit tests
    integration: Integration tests (requires Docker)
    slow: Slow tests (>10s)
```

### Test Database Fixture

```python
# tests/conftest.py
import pytest
import psycopg2
from psycopg2.extras import RealDictCursor
import os

@pytest.fixture(scope="session")
def pg_dsn():
    return f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"

@pytest.fixture(scope="function")
def db_conn(pg_dsn):
    conn = psycopg2.connect(pg_dsn, cursor_factory=RealDictCursor)
    conn.autocommit = False
    yield conn
    conn.rollback()
    conn.close()

@pytest.fixture(scope="function")
def clean_db(db_conn):
    """Truncate test tables before each test."""
    with db_conn.cursor() as cur:
        cur.execute("TRUNCATE fact_chunks, fact_article_authors, fact_articles, dim_content, dim_author, dim_time, dim_source RESTART IDENTITY CASCADE")
    db_conn.commit()
```

### Sample Unit Tests

```python
# tests/test_etl.py
import pytest
from etl.etl_warehouse import ETLWarehouse, clean_html, chunk_text

class TestETL:
    def test_clean_html_removes_tags(self):
        html = "<p>Hello <b>world</b></p><script>alert('xss')</script>"
        assert clean_html(html) == "Hello world"

    def test_chunk_text_basic(self):
        text = " ".join(["word"] * 600)
        chunks = chunk_text(text, chunk_size=500, overlap=50)
        assert len(chunks) == 2
        assert len(chunks[0].split()) == 500
        assert len(chunks[1].split()) == 150  # 600 - 500 + 50 overlap

    def test_chunk_text_short(self):
        text = "Short text"
        chunks = chunk_text(text)
        assert chunks == ["Short text"]

# tests/test_vectorize.py
class TestVectorize:
    @pytest.mark.integration
    def test_embedding_generation(self):
        from vectorize.vectorize import get_embedding
        vec = get_embedding("Test article about technology")
        assert len(vec) == 384  # BGE-small
        assert all(isinstance(x, float) for x in vec)
```

### Run Tests

```bash
# Unit tests only
pytest -m unit

# Integration tests (requires Docker services)
docker compose up -d
pytest -m integration

# All tests with coverage
pytest --cov=etl --cov=vectorize --cov=search --cov-report=term-missing
```

## 2. RAG Evaluation with RAGAS

### Why RAGAS?

RAGAS evaluates RAG systems using LLM-as-judge across 4 metrics:

| Metric | What it measures | Target |
|--------|------------------|--------|
| **Faithfulness** | Answer grounded in retrieved context | > 0.8 |
| **Answer Relevancy** | Answer addresses the question | > 0.8 |
| **Context Precision** | Relevant chunks ranked higher | > 0.7 |
| **Context Recall** | All relevant chunks retrieved | > 0.7 |

### Installation

```bash
pip install ragas langchain-openai datasets
```

### Evaluation Script

```python
# tests/test_ragas_evaluation.py
import os
import json
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from ragas.llms import LangchainLLMWrapper
from langchain_openai import ChatOpenAI
from search.engine import Pipeline

# Use Groq-compatible endpoint or local model for evaluation
os.environ["OPENAI_API_KEY"] = os.getenv("GROQ_API_KEY")
os.environ["OPENAI_API_BASE"] = "https://api.groq.com/openai/v1"

# Initialize RAGAS with Groq LLM
eval_llm = LangchainLLMWrapper(
    ChatOpenAI(
        model="llama-3.1-8b-instant",
        temperature=0,
        base_url="https://api.groq.com/openai/v1",
        api_key=os.getenv("GROQ_API_KEY"),
    )
)

# Golden test set
TEST_QUESTIONS = [
    {
        "question": "Tóm tắt tin tức về kinh tế Việt Nam hôm nay",
        "ground_truth": "GDP quý 4 tăng 6.5%, xuất khẩu tăng trưởng...",
    },
    {
        "question": "Ai là tác giả bài viết về công nghệ mới nhất?",
        "ground_truth": "Nguyễn Văn A",
    },
    {
        "question": "So sánh quan điểm của VnExpress và Thanh Niên về chính sách mới",
        "ground_truth": "VnExpress tập trung vào..., Thanh Niên nhấn mạnh...",
    },
]

def run_ragas_evaluation():
    """Run RAGAS evaluation on current pipeline."""
    pipeline = Pipeline()
    results = []

    for test in TEST_QUESTIONS:
        # Get RAG response
        response = pipeline.ask(test["question"])
        
        results.append({
            "question": test["question"],
            "answer": response.summary,
            "contexts": [r.content for r in response.results],
            "ground_truth": test["ground_truth"],
        })

    # Create dataset
    dataset = Dataset.from_list(results)

    # Evaluate
    score = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
        llm=eval_llm,
    )

    print("\n=== RAGAS Results ===")
    print(f"Faithfulness:       {score['faithfulness']:.3f}")
    print(f"Answer Relevancy:   {score['answer_relevancy']:.3f}")
    print(f"Context Precision:  {score['context_precision']:.3f}")
    print(f"Context Recall:     {score['context_recall']:.3f}")

    return score

if __name__ == "__main__":
    run_ragas_evaluation()
```

### Expected Output

```
=== RAGAS Results ===
Faithfulness:       0.72
Answer Relevancy:   0.68
Context Precision:  0.79
Context Recall:     0.61
```

### Improving Scores

| Low Score | Action |
|-----------|--------|
| Faithfulness | Improve prompt: "Chỉ trả lời dựa trên ngữ cảnh, không bịa đặt" |
| Answer Relevancy | Add "Trả lời trực tiếp, ngắn gọn" to prompt |
| Context Precision | Tune HNSW `ef_search`, increase top-K, add metadata filters |
| Context Recall | Better chunking (semantic), hybrid search (BM25 + vector) |

## 3. Load Testing with Locust

### locustfile.py

```python
from locust import HttpUser, task, between
import json
import random

QUERIES = [
    "Tóm tắt tin tức kinh tế hôm nay",
    "Công nghệ AI mới nhất là gì",
    "Thời tiết Hà Nội tuần này",
    "Bóng đá V-League kết quả",
    "Chính sách thuế mới 2024",
]

class RAGUser(HttpUser):
    wait_time = between(1, 3)
    host = os.getenv("RAG_API_URL", "https://xxx.execute-api.ap-southeast-2.amazonaws.com/prod")

    @task(3)
    def ask_question(self):
        query = random.choice(QUERIES)
        self.client.post("/ask", json={
            "query": query,
            "model": "qwen3-8b-instant",
            "top_k": 5
        })

    @task(1)
    def ask_complex(self):
        self.client.post("/ask", json={
            "query": "So sánh quan điểm các báo về chính sách đất đai mới",
            "model": "gemini-2.0-flash",
            "top_k": 10
        })
```

### Run Load Test

```bash
# Install
pip install locust

# Run (headless, 50 users, 2 min)
locust -f locustfile.py --headless -u 50 -r 5 --run-time 2m --html report.html

# Or with UI
locust -f locustfile.py
# Open http://localhost:8089
```

### Target Metrics

| Metric | Target |
|--------|--------|
| p50 Latency | < 3s |
| p95 Latency | < 8s |
| p99 Latency | < 15s |
| Error Rate | < 0.1% |
| Throughput | > 10 RPS |

## 4. Production Monitoring

### CloudWatch Dashboards

Create dashboard with these widgets:

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Invocations", "FunctionName", "newsrag-rag-api"],
          [".", "Errors", ".", "."],
          [".", "Duration", ".", "."],
          [".", "Throttles", ".", "."]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "ap-southeast-2",
        "title": "RAG API - Invocations & Errors"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Duration", "FunctionName", "newsrag-etl", { "stat": "p50" }],
          [".", ".", ".", ".", { "stat": "p95" }],
          [".", ".", ".", ".", { "stat": "p99" }]
        ],
        "title": "ETL Lambda Duration (p50/p95/p99)"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "newsrag-news-raw"],
          [".", "ApproximateNumberOfMessagesNotVisible", ".", "."],
          [".", "NumberOfMessagesSent", ".", "."],
          [".", "NumberOfMessagesReceived", ".", "."]
        ],
        "title": "SQS Queue Depth & Throughput"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/RDS", "CPUUtilization", "DBClusterIdentifier", "newsrag-postgres"],
          [".", "DatabaseConnections", ".", "."],
          [".", "FreeableMemory", ".", "."]
        ],
        "title": "Aurora Serverless v2 Metrics"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Bedrock", "Invocations", "ModelId", "amazon.titan-embed-text-v2:0"],
          [".", "InvocationLatency", ".", "."],
          [".", "Throttles", ".", "."]
        ],
        "title": "Bedrock Titan Embeddings"
      }
    }
  ]
}
```

### CloudWatch Alarms

```hcl
# Terraform alarms
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "newsrag-rag-api-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "RAG API Lambda errors detected"
  
  dimensions = {
    FunctionName = aws_lambda_function.rag_api.function_name
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "sqs_backlog" {
  alarm_name          = "newsrag-sqs-backlog"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Average"
  threshold           = 100
  alarm_description   = "SQS queue backing up - consumer may be down"
  
  dimensions = {
    QueueName = aws_sqs_queue.news_raw.name
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "rag_latency" {
  alarm_name          = "newsrag-rag-api-high-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "p95"
  threshold           = 20000  # 20 seconds in ms
  alarm_description   = "RAG API p95 latency > 20s"
  
  dimensions = {
    FunctionName = aws_lambda_function.rag_api.function_name
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### Structured Logging

```python
# In Lambda functions - use structured JSON logs
import logging
import json

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "function": record.funcName,
            "line": record.lineno,
        }
        if hasattr(record, "extra_fields"):
            log_data.update(record.extra_fields)
        return json.dumps(log_data)

# Usage
logger = logging.getLogger()
logger.handlers[0].setFormatter(JsonFormatter())

# Log with context
logger.info("ETL processing article", extra={"extra_fields": {
    "article_id": article_id,
    "source": source_name,
    "chunks": len(chunks),
    "duration_ms": duration_ms
}})
```

### Log Insights Queries

```sql
-- Find slow ETL invocations
fields @timestamp, @duration, @message
| filter @logStream like /etl/
| sort @duration desc
| limit 20

-- Error patterns
fields @timestamp, @message
| filter @message like /ERROR|Exception|Traceback/
| sort @timestamp desc
| limit 50

-- Bedrock throttling
fields @timestamp, @message
| filter @message like /Throttling|RateExceeded/
| sort @timestamp desc
```

## 5. End-to-End Test (Playwright)

```python
# tests/e2e/test_chat.py
from playwright.sync_api import Page, expect
import pytest

@pytest.mark.e2e
def test_chat_flow(page: Page):
    page.goto("https://your-dashboard.vercel.app/chat")
    
    # Wait for page load
    expect(page.locator("h1")).to_contain_text("AI Chat")
    
    # Select model
    page.locator("select").select_option("qwen3-8b-instant")
    
    # Send question
    page.fill("input[placeholder*='Hỏi']", "Tóm tắt tin tức kinh tế hôm nay")
    page.click("button:has-text('Send')")
    
    # Wait for response
    expect(page.locator("text=Tóm tắt")).to_be_visible(timeout=30000)
    
    # Verify sources shown
    expect(page.locator("text=Sources")).to_be_visible()
    
    # Click source link
    page.locator("details summary").first.click()
    expect(page.locator("li").first).to_be_visible()
```

```bash
# Run E2E tests
npm install -g playwright
playwright install
pytest tests/e2e/ -v
```

## CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/test.yml
name: Test & Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg15
        env:
          POSTGRES_DB: newsrag
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports: [5432:5432]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pip install pytest pytest-cov
      - run: pytest -m "unit or integration" --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v3

  ragas-eval:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install ragas langchain-openai datasets
      - run: python tests/test_ragas_evaluation.py
      - name: Comment PR with results
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = fs.readFileSync('ragas_results.json', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## RAGAS Evaluation Results\n\`\`\`json\n${results}\n\`\`\``
            })

  deploy:
    needs: [test, ragas-eval]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-2
      - name: Deploy Infrastructure
        run: |
          terraform init
          terraform apply -auto-approve
      - name: Build & Push Docker
        run: |
          docker build -t news-crawler .
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
          docker tag news-crawler:latest $ECR_URI/newsrag-api:latest
          docker push $ECR_URI/newsrag-api:latest
      - name: Deploy Lambdas
        run: ./deploy.sh
```

---

**Next:** [Cost Optimization](5.16-Cost/)