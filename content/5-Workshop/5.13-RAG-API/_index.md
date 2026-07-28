---
title: "RAG API (Lambda + API Gateway)"
date: 2026-07-28
weight: 13
chapter: false
pre: " <b> 5.13 </b> "
---

# RAG API: Lambda + API Gateway

This section covers the REST API that accepts natural language queries, retrieves relevant chunks via pgvector similarity search, and generates answers using LLMs (Groq/Gemini).

## Architecture

```
Client (Next.js/FastAPI)
       │
       ▼
API Gateway (REST, /ask endpoint)
       │
       ▼
Lambda RAG API (Python 3.11, 30s timeout, 1024 MB)
       │
       ├────► Bedrock Titan Embed v2 (embed query)
       │
       ├────► Aurora pgvector (HNSW similarity search)
       │
       └────► Groq / Gemini API (generate answer)
       │
       ▼
Client (JSON response with answer + sources)
```

## Request/Response Format

### Request
```json
POST /ask
Content-Type: application/json

{
  "query": "Tóm tắt tin tức về kinh tế Việt Nam hôm nay",
  "model": "qwen3-8b-instant",      // optional: qwen3-8b-instant, llama-3.1-8b-instant, gemini-2.0-flash
  "top_k": 5,                        // optional: default 5
  "is_vanilla": false                // optional: true = no RAG context, just LLM
}
```

### Response
```json
{
  "query": "Tóm tắt tin tức về kinh tế Việt Nam hôm nay",
  "summary": "Hôm nay, GDP quý 4 năm 2023 tăng 6.5%... [trả lời tiếng Việt dựa trên ngữ cảnh]",
  "results": [
    {
      "chunk_id": 12345,
      "article_id": 678,
      "content": "GDP quý 4 tăng 6.5% so với cùng kỳ...",
      "title": "GDP quý 4 tăng 6.5%",
      "source_name": "VnExpress",
      "source_domain": "vnexpress.net",
      "published_at": "2024-01-15T08:30:00+07:00",
      "score": 0.89
    },
    ...
  ],
  "total": 5,
  "duration_ms": 1250.5
}
```

## Lambda Function Code

### File Structure
```
deploy/
├── rag/
│   ├── lambda_function.py
│   ├── requirements.txt
│   ├── retriever.py
│   ├── generator.py
│   └── schemas.py
```

### schemas.py
```python
from typing import List, Optional
from pydantic import BaseModel, Field


class SearchHit(BaseModel):
    chunk_id: int
    article_id: int
    content: str
    title: str
    source_name: str
    source_domain: str
    published_at: str
    score: float


class GeneratorResponse(BaseModel):
    query: str
    summary: str
    results: List[SearchHit]
    total: int
    duration_ms: float


class AskRequest(BaseModel):
    query: str
    model: Optional[str] = "qwen3-8b-instant"
    top_k: Optional[int] = 5
    is_vanilla: Optional[bool] = False
```

### retriever.py
```python
import os
import logging
from typing import List
import psycopg2
from psycopg2.extras import RealDictCursor
from bedrock_utils import embed_text, EMBEDDING_DIM

logger = logging.getLogger(__name__)


def get_db_connection():
    return psycopg2.connect(
        host=os.environ["DB_HOST"],
        port=os.environ["DB_PORT"],
        database=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
        cursor_factory=RealDictCursor,
    )


def retrieve(query: str, top_k: int = 5) -> List[dict]:
    """
    Embed query with Bedrock, search pgvector for similar chunks.
    Returns list of dicts with chunk data + similarity score.
    """
    # 1. Embed query
    query_vec = embed_text(query)
    
    # 2. Search Aurora pgvector
    conn = get_db_connection()
    try:
        with conn.cursor() as cur:
            # Set HNSW ef_search for better recall (default 40)
            cur.execute("SET LOCAL hnsw.ef_search = 64")
            
            # Vector similarity search with metadata join
            cur.execute("""
                SELECT
                    fc.chunk_id,
                    fc.article_id,
                    fc.content,
                    fc.chunk_index,
                    dc.title,
                    ds.source_name,
                    ds.source_domain,
                    dt.full_datetime as published_at,
                    1 - (fc.embedding <=> %s::vector) as similarity
                FROM fact_chunks fc
                JOIN fact_articles fa ON fc.article_id = fa.article_id
                JOIN dim_content dc ON fa.content_id = dc.content_id
                JOIN dim_source ds ON fa.source_id = ds.source_id
                JOIN dim_time dt ON fa.time_id = dt.time_id
                WHERE fc.embedding IS NOT NULL
                ORDER BY fc.embedding <=> %s::vector
                LIMIT %s
            """, (query_vec, query_vec, top_k))
            
            results = cur.fetchall()
            
    finally:
        conn.close()
    
    logger.info(f"Retrieved {len(results)} chunks for query: {query[:50]}...")
    return results
```

### generator.py
```python
import os
import json
import logging
import requests
from typing import List, Dict, Optional
from schemas import SearchHit, AskRequest, GeneratorResponse

logger = logging.getLogger(__name__)


# Model configurations
MODELS = {
    "qwen3-8b-instant": {
        "provider": "groq",
        "model_id": "qwen/qwen3-8b-instant",
        "api_key_env": "GROQ_API_KEY",
        "api_url": "https://api.groq.com/openai/v1/chat/completions",
    },
    "llama-3.1-8b-instant": {
        "provider": "groq",
        "model_id": "meta-llama/llama-3.1-8b-instant",
        "api_key_env": "GROQ_API_KEY",
        "api_url": "https://api.groq.com/openai/v1/chat/completions",
    },
    "gemini-2.0-flash": {
        "provider": "google",
        "model_id": "gemini-2.0-flash",
        "api_key_env": "GEMINI_API_KEY",
        "api_url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent",
    },
}


def build_prompt(query: str, hits: List[SearchHit], is_vanilla: bool = False) -> List[dict]:
    """Build messages for LLM."""
    
    if is_vanilla or not hits:
        return [
            {"role": "system", "content": "Bạn là một trợ lý AI hữu ích. Trả lời bằng tiếng Việt."},
            {"role": "user", "content": query}
        ]
    
    # Build context from retrieved chunks
    context_parts = []
    for i, hit in enumerate(hits, 1):
        context_parts.append(
            f"[Nguồn {i}] {hit.source_name} ({hit.source_domain}) - {hit.published_at}\n"
            f"Tiêu đề: {hit.title}\n"
            f"Nội dung: {hit.content[:500]}..."
        )
    
    context = "\n\n".join(context_parts)
    
    return [
        {
            "role": "system",
            "content": (
                "Bạn là một trợ lý AI phân tích tin tức. "
                "Trả lời câu hỏi dựa CHỈ trên ngữ cảnh tin tức được cung cấp. "
                "Nếu ngữ cảnh không đủ thông tin, hãy nói rõ. "
                "Trả lời bằng tiếng Việt, súc tích, chính xác. "
                "Luôn trích dẫn nguồn [Nguồn X] trong câu trả lời."
            )
        },
        {
            "role": "user",
            "content": f"Ngữ cảnh tin tức:\n{context}\n\nCâu hỏi: {query}"
        }
    ]


def call_groq(model_config: dict, messages: List[dict]) -> str:
    """Call Groq API (OpenAI-compatible)."""
    api_key = os.environ[model_config["api_key_env"]]
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    payload = {
        "model": model_config["model_id"],
        "messages": messages,
        "temperature": 0.3,
        "max_tokens": 1024,
        "stream": False
    }
    
    resp = requests.post(model_config["api_url"], headers=headers, json=payload, timeout=30)
    resp.raise_for_status()
    return resp.json()["choices"][0]["message"]["content"]


def call_gemini(model_config: dict, messages: List[dict]) -> str:
    """Call Google Gemini API."""
    api_key = os.environ[model_config["api_key_env"]]
    url = f"{model_config['api_url']}?key={api_key}"
    
    # Convert messages to Gemini format
    contents = []
    for msg in messages:
        role = "user" if msg["role"] == "user" else "model"
        contents.append({"role": role, "parts": [{"text": msg["content"]}]})
    
    payload = {"contents": contents}
    
    resp = requests.post(url, json=payload, timeout=30)
    resp.raise_for_status()
    return resp.json()["candidates"][0]["content"]["parts"][0]["text"]


def generate_answer(request: AskRequest, hits: List[SearchHit]) -> GeneratorResponse:
    import time
    start = time.time()
    
    model_name = request.model or "qwen3-8b-instant"
    model_config = MODELS.get(model_name, MODELS["qwen3-8b-instant"])
    
    messages = build_prompt(request.query, hits, request.is_vanilla)
    
    try:
        if model_config["provider"] == "groq":
            answer = call_groq(model_config, messages)
        elif model_config["provider"] == "google":
            answer = call_gemini(model_config, messages)
        else:
            raise ValueError(f"Unknown provider: {model_config['provider']}")
    except Exception as e:
        logger.exception(f"LLM call failed: {e}")
        # Fallback to Gemini if Groq fails
        if model_config["provider"] == "groq":
            logger.info("Falling back to Gemini...")
            gemini_config = MODELS["gemini-2.0-flash"]
            answer = call_gemini(gemini_config, messages)
        else:
            raise
    
    duration_ms = (time.time() - start) * 1000
    
    return GeneratorResponse(
        query=request.query,
        summary=answer,
        results=hits,
        total=len(hits),
        duration_ms=round(duration_ms, 2)
    )
```

### lambda_function.py
```python
import json
import logging
import os
import time
from typing import Dict, Any

from schemas import AskRequest, SearchHit
from retriever import retrieve
from generator import generate_answer

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def lambda_handler(event: Dict[str, Any], context) -> Dict[str, Any]:
    """API Gateway proxy integration handler."""
    logger.info(f"Event: {json.dumps(event)}")
    
    try:
        # Parse request
        if event.get("body"):
            body = json.loads(event["body"])
        else:
            body = event
        
        request = AskRequest(**body)
        
        # Retrieve relevant chunks
        hits_data = retrieve(request.query, request.top_k)
        hits = [SearchHit(**h) for h in hits_data]
        
        # Generate answer
        response = generate_answer(request, hits)
        
        return {
            "statusCode": 200,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*",
                "Access-Control-Allow-Headers": "Content-Type",
                "Access-Control-Allow-Methods": "POST,OPTIONS"
            },
            "body": response.model_dump_json()
        }
    
    except Exception as e:
        logger.exception(f"RAG API error: {e}")
        return {
            "statusCode": 500,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            "body": json.dumps({
                "error": "Internal server error",
                "message": str(e)
            })
        }
```

### requirements.txt
```txt
psycopg2-binary==2.9.9
pydantic==2.5.3
requests==2.31.0
boto3==1.34.0
botocore==1.34.0
```

## Terraform Deployment

### API Gateway + Lambda

```hcl
# Lambda Function
resource "aws_lambda_function" "rag_api" {
  function_name = "newsrag-rag-api"
  description   = "RAG Query API: embed query -> pgvector search -> LLM generate"
  
  runtime       = "python3.11"
  handler       = "lambda_function.lambda_handler"
  timeout       = 30
  memory_size   = 1024
  
  filename         = "rag.zip"
  source_code_hash = filebase64sha256("rag.zip")
  
  vpc_config {
    subnet_ids         = [aws_subnet.pub_a.id, aws_subnet.pub_b.id]
    security_group_ids = [aws_security_group.lambda_sg.id]
  }
  
  role = aws_iam_role.lambda_role.arn
  
  environment {
    variables = {
      DB_HOST     = aws_rds_cluster.main.endpoint
      DB_PORT     = "5432"
      DB_NAME     = "newsrag"
      DB_USER     = var.db_user
      DB_PASSWORD = var.db_password
      GROQ_API_KEY    = var.groq_api_key
      GEMINI_API_KEY  = var.gemini_api_key
      AWS_REGION      = "ap-southeast-2"
    }
  }
}

# API Gateway REST API
resource "aws_api_gateway_rest_api" "rag" {
  name        = "newsrag-api"
  description = "News RAG Query API"
  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

# Resource /ask
resource "aws_api_gateway_resource" "ask" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  parent_id   = aws_api_gateway_rest_api.rag.root_resource_id
  path_part   = "ask"
}

# POST method
resource "aws_api_gateway_method" "ask_post" {
  rest_api_id   = aws_api_gateway_rest_api.rag.id
  resource_id   = aws_api_gateway_resource.ask.id
  http_method   = "POST"
  authorization = "NONE"
}

# Lambda integration
resource "aws_api_gateway_integration" "ask_lambda" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  resource_id = aws_api_gateway_resource.ask.id
  http_method = aws_api_gateway_method.ask_post.http_method
  
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.rag_api.invoke_arn
}

# OPTIONS method for CORS
resource "aws_api_gateway_method" "ask_options" {
  rest_api_id   = aws_api_gateway_rest_api.rag.id
  resource_id   = aws_api_gateway_resource.ask.id
  http_method   = "OPTIONS"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "ask_options" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  resource_id = aws_api_gateway_resource.ask.id
  http_method = aws_api_gateway_method.ask_options.http_method
  type        = "MOCK"
  
  request_templates = {
    "application/json" = '{"statusCode": 200}'
  }
}

resource "aws_api_gateway_method_response" "ask_200" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  resource_id = aws_api_gateway_resource.ask.id
  http_method = aws_api_gateway_method.ask_post.http_method
  status_code = "200"
  
  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin" = true
  }
}

resource "aws_api_gateway_method_response" "ask_options_200" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  resource_id = aws_api_gateway_resource.ask.id
  http_method = aws_api_gateway_method.ask_options.http_method
  status_code = "200"
  
  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin" = true
    "method.response.header.Access-Control-Allow-Methods" = true
    "method.response.header.Access-Control-Allow-Headers" = true
  }
}

resource "aws_api_gateway_integration_response" "ask_options" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  resource_id = aws_api_gateway_resource.ask.id
  http_method = aws_api_gateway_method.ask_options.http_method
  status_code = aws_api_gateway_method_response.ask_options_200.status_code
  
  response_parameters = {
    "method.response.header.Access-Control-Allow-Origin" = "'*'"
    "method.response.header.Access-Control-Allow-Methods" = "'POST,OPTIONS'"
    "method.response.header.Access-Control-Allow-Headers" = "'Content-Type,Authorization'"
  }
}

# Deployment
resource "aws_api_gateway_deployment" "rag" {
  rest_api_id = aws_api_gateway_rest_api.rag.id
  stage_name  = "prod"
  
  triggers = {
    redeployment = sha256(jsonencode([
      aws_api_gateway_resource.ask.id,
      aws_api_gateway_method.ask_post.id,
      aws_api_gateway_integration.ask_lambda.id,
    ]))
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Lambda permission for API Gateway
resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.rag_api.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.rag.execution_arn}/*/*"
}

# Output API URL
output "api_url" {
  value = "${aws_api_gateway_deployment.rag.invoke_url}/ask"
}
```

## Testing the API

### 1. Get API URL
```bash
API_URL=$(terraform output -raw api_url)
echo $API_URL
# https://xxxxx.execute-api.ap-southeast-2.amazonaws.com/prod/ask
```

### 2. Test with curl
```bash
curl -X POST "$API_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Tóm tắt tin tức về kinh tế Việt Nam hôm nay",
    "model": "qwen3-8b-instant",
    "top_k": 5
  }' | jq .
```

### 3. Test vanilla (no RAG)
```bash
curl -X POST "$API_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Việt Nam có bao nhiêu tỉnh thành?",
    "model": "gemini-2.0-flash",
    "is_vanilla": true
  }' | jq .
```

### 4. Test from Next.js Frontend
```typescript
// lib/api.ts
export async function askRAG(query: string, model = "qwen3-8b-instant", topK = 5) {
  const res = await fetch(`${process.env.NEXT_PUBLIC_RAG_API_URL}/ask`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, model, top_k: topK })
  });
  
  if (!res.ok) throw new Error("API error");
  return res.json();
}
```

## Monitoring

### CloudWatch Logs
```bash
aws logs tail /aws/lambda/newsrag-rag-api --follow
```

### Key Metrics to Alert
| Metric | Threshold |
|--------|-----------|
| `Duration` (p95) | > 20s |
| `Errors` | > 0 in 5min |
| `Throttles` | > 0 |
| `Invocations` | = 0 for 1 hour (if expected traffic) |

## Cost Estimation

| Component | Usage | Monthly Cost |
|-----------|-------|--------------|
| Lambda (1M req, 1GB, 500ms avg) | 1M × 500ms × 1GB | ~$3.50 |
| API Gateway (1M req) | 1M requests | ~$3.50 |
| Bedrock Titan Embed (query) | 1M queries × 50 tokens | ~$0.50 |
| Groq API (free tier) | 1M req | $0 (free tier) |
| Aurora (query only) | Included in DB cost | - |
| **Total** | | **~$7.50/month** |

## Next Steps

After RAG API works:
1. **Frontend Integration** - Next.js Dashboard: [Frontend](5.14-Frontend/)
2. **Testing & Evaluation** - RAGAS metrics: [Testing](5.15-Testing/)
3. **Cost Optimization** - Serverless tuning: [Cost](5.16-Cost/)

---

**Next:** [Frontend Integration (Next.js + FastAPI)](5.14-Frontend/)