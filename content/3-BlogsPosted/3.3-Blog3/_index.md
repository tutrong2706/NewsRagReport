---
title: "Blog 3: Building an Optimized RAG System"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Realizing the RAG System with Amazon Bedrock and Aurora pgvector

**Retrieval-Augmented Generation (RAG)** is a method that combines the power of Large Language Models (LLMs) with a business's proprietary data. For RAG to work effectively, the heart of the system is the **Embedding Model** (to convert text into vectors) and the **Vector Database** (to store and search vectors). 

Here is how we built this heart entirely using the AWS ecosystem.

### 1. Amazon Bedrock: The Power of Embedding

Instead of self-hosting models like HuggingFace SentenceTransformer on EC2 (which is very expensive for GPUs) or Lambda (which runs slowly and suffers from cold starts), we used **Amazon Bedrock** with the `amazon.titan-embed-text-v2:0` model.

*   **Performance and Cost**: The Bedrock API responds incredibly fast, processing embeddings for text chunks in milliseconds. The cost is calculated per token, which is extremely cheap for small to medium projects.
*   **Absolute Consistency**: A vital rule of RAG is that the data in the Database and the user's question (Query) must be converted into vectors by the **same model**. By calling the Bedrock API during both the ETL phase and the RAG API phase, we ensure the absolute consistency of our vector space.

### 2. Amazon Aurora Serverless v2 + pgvector

Instead of using third-party Vector Databases like Qdrant, Pinecone, or Milvus, we decided to use **Amazon Aurora PostgreSQL** combined with the **`pgvector`** extension.

*   **All-in-one**: PostgreSQL allows storing metadata (like author, publish date, URL) in a relational structure (Star Schema) combined with a `vector` column to store embeddings. This makes it easy for us to perform hybrid queries: "Find articles about the economy (vector search) but only fetch those published this week (SQL filter)".
*   **HNSW Indexing**: To search quickly across hundreds of thousands of data chunks, we configured the **HNSW (Hierarchical Navigable Small World)** index on the vector column. Consequently, cosine similarity search queries return results almost instantaneously.
*   **Serverless v2**: The Aurora Database automatically scales up and down (from 0.5 to 2 ACUs) based on system load, optimizing costs while ensuring high performance when users query the Q&A system.

### Conclusion

The architectural combination of Amazon Bedrock and Aurora pgvector provides a closed-loop (End-to-End) RAG solution entirely on AWS. The news data never has to leave the system's VPC to make external calls, ensuring maximum security and reducing network latency. This is the technical highlight we are most proud of in the entire application development process.