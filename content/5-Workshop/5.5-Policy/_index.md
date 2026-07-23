---
title : "RAG API Module"
date : 2024-01-01 
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

#### Intelligent Assistant with News-based Q&A

The RAG (Retrieval-Augmented Generation) module is the heart of the system, combining the semantic search capabilities of Qdrant/pgvector and the reasoning power of LLMs to answer user questions based on actual news.

**1. RAG Processing Flow**

When a user asks a question (e.g., *"What is the situation with Typhoon Yagi today?"*), the system executes:
1. **Query Embedding**: Transforms the question into a 1024-dimensional vector using Amazon Bedrock.
2. **Vector Search (Retrieval)**: Uses cosine similarity to find the Top-K (e.g., 5) most relevant text chunks from Qdrant/pgvector.
3. **Prompt Engineering**: Combines these Top-K chunks into a "Context" along with the user's question into the Prompt.
4. **LLM Generation**: Sends the Prompt to Groq (Qwen3-8B) or Gemini Flash to generate a natural answer with source citations.

**2. System Prompt Configuration**

To prevent LLM hallucinations or fabrication of information, the Prompt is strictly designed:

```python
# search/prompts.py
NEWS_RAG_SYSTEM_PROMPT = """You are a Senior News Analyst, capable of professional, honest, and objective analysis.
Response style:
- Objective, honest, based ENTIRELY on the provided data. Absolutely do not use external knowledge to fabricate information.
- Always cite sources clearly and accurately (e.g.: According to the article [Article Title]).
- If the CONTEXT has absolutely no relevant information, reply: "Based on the provided documents, there is not enough information to accurately answer about..."
"""
```

**3. Multi-LLM Architecture**

The system is configured to support multiple different LLMs for fallback purposes or quality comparison:
- **Model 1 (Primary)**: Groq API with `qwen-2.5-32b` or `llama-3.1-8b-instant` — ultra-fast speed.
- **Model 2 (Fallback)**: Google Gemini `gemini-2.0-flash-exp` — excellent long-context handling.

**4. RAG Quality Evaluation (Ragas)**

The quality of the RAG pipeline is measured using the Ragas framework with metrics:
- **Context Precision**: Are the most relevant documents ranked highly?
- **Context Recall**: Did the system retrieve enough information to answer?
- **Faithfulness**: Is the LLM faithful to the context (or hallucinating)?
- **Answer Relevancy**: Is the answer focused on the question?
