---
title : "ETL & Vectorize Module"
date : 2024-01-01 
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

#### Data Normalization and Vector Embedding

Raw data from the Crawler needs to be cleaned, chunked, and transformed into vectors to support semantic search.

**1. Data Cleaning & Star Schema**

Data is stripped of redundant HTML tags, date formats are normalized, and stored in RDS Aurora PostgreSQL following the Star Schema model:

- `dim_source`: Contains newspaper information
- `dim_time`: Detailed split of day/month/year
- `dim_author`: Manages normalized author information
- `fact_articles`: Central table containing articles
- `fact_chunks`: Contains text chunks split from articles

**2. Chunking with Langchain**

Articles are split into chunks with a maximum length of 800 characters and a 150-character overlap to preserve context between segments.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=150,
    separators=["\n\n", "\n", ".", " ", ""]
)

chunks = text_splitter.split_text(article_content)
```

**3. Vector Embedding with Amazon Bedrock**

NewsRAG v2 uses Amazon Bedrock with the `amazon.titan-embed-text-v2:0` model (1024 dimensions) to generate vectors instead of hosting a local `BAAI/bge-m3` model like in v1. This reduces container load and saves resources.

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

Vectors are upserted into Qdrant Cloud or stored directly in Aurora via the `pgvector` extension with HNSW (Hierarchical Navigable Small World) index to ensure `Top-K` query speeds remain under 50ms even with millions of vectors.
