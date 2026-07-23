---
title : "Module ETL & Vectorize"
date : 2024-01-01 
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

#### Chuẩn hóa Dữ liệu và Vector Embedding

Dữ liệu thô từ Crawler cần được làm sạch, cắt nhỏ (chunking) và biến đổi thành vector để phục vụ cho tìm kiếm ngữ nghĩa (semantic search).

**1. Data Cleaning & Star Schema**

Dữ liệu được làm sạch các thẻ HTML thừa, chuẩn hóa định dạng ngày tháng và lưu trữ vào RDS Aurora PostgreSQL theo mô hình Star Schema:

- `dim_source`: Chứa thông tin về báo điện tử
- `dim_time`: Phân tách chi tiết ngày/tháng/năm
- `dim_author`: Quản lý thông tin tác giả đã được chuẩn hóa
- `fact_articles`: Bảng trung tâm chứa bài viết
- `fact_chunks`: Chứa các đoạn văn bản (chunks) được cắt từ bài viết

**2. Chunking với Langchain**

Bài viết được cắt nhỏ thành các chunk có độ dài tối đa 800 ký tự với độ chênh lệch (overlap) là 150 ký tự để giữ trọn vẹn ngữ cảnh giữa các đoạn.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=150,
    separators=["\n\n", "\n", ".", " ", ""]
)

chunks = text_splitter.split_text(article_content)
```

**3. Vector Embedding với Amazon Bedrock**

NewsRAG v2 sử dụng Amazon Bedrock với mô hình `amazon.titan-embed-text-v2:0` (1024 chiều) để tạo vector thay vì host mô hình local `BAAI/bge-m3` như ở v1, giúp giảm tải container và tiết kiệm tài nguyên.

```python
import boto3
import json

bedrock_runtime = boto3.client('bedrock-runtime', region_name='ap-southeast-2')

def get_embedding(text):
    body = json.dumps({"inputText": text})
    response = bedrock_runtime.invoke_model(
        modelId='amazon.titan-embed-text-v2:0',
        contentType='application/json',
        accept='application/json',
        body=body
    )
    response_body = json.loads(response.get('body').read())
    return response_body.get('embedding')
```

**4. Vector Database (Qdrant & pgvector)**

Các vectors được upsert vào Qdrant Cloud hoặc lưu trữ trực tiếp trong Aurora thông qua extension `pgvector` với index HNSW (Hierarchical Navigable Small World) để đảm bảo tốc độ truy vấn `Top-K` luôn dưới 50ms ngay cả với hàng triệu vectors.
