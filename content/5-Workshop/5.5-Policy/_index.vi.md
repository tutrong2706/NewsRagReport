---
title : "Module RAG API"
date : 2024-01-01 
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

#### Trợ lý Thông minh với Hỏi đáp dựa trên Tin tức

Module RAG (Retrieval-Augmented Generation) là trái tim của hệ thống, kết hợp khả năng tìm kiếm ngữ nghĩa của Qdrant/pgvector và sức mạnh suy luận của LLM để trả lời câu hỏi người dùng dựa trên tin tức thực tế.

**1. Luồng xử lý RAG**

Khi người dùng đặt câu hỏi (VD: *"Tình hình bão Yagi hôm nay thế nào?"*), hệ thống thực hiện các bước:
1. **Query Embedding**: Biến đổi câu hỏi thành vector 1024 chiều bằng Amazon Bedrock.
2. **Vector Search (Retrieval)**: So sánh cosine similarity để tìm ra Top-K (VD: 5) chunks văn bản liên quan nhất từ Qdrant/pgvector.
3. **Prompt Engineering**: Kết hợp Top-K chunks này thành "Context" cùng với câu hỏi của người dùng đưa vào Prompt.
4. **LLM Generation**: Gửi Prompt cho Groq (Qwen3-8B) hoặc Gemini Flash để sinh ra câu trả lời tự nhiên, có trích dẫn nguồn.

**2. Cấu hình System Prompt**

Để tránh tình trạng LLM bị ảo giác (hallucination) hoặc tự bịa thông tin, Prompt được thiết kế chặt chẽ:

```python
# search/prompts.py
NEWS_RAG_SYSTEM_PROMPT = """Bạn là một Chuyên gia Phân tích Tin tức cấp cao, có khả năng phân tích chuyên nghiệp, trung thực và khách quan.
Phong cách trả lời:
- Khách quan, trung thực, dựa HOÀN TOÀN trên dữ liệu được cung cấp. Tuyệt đối không dùng kiến thức ngoài để tự bịa thêm.
- Luôn trích dẫn nguồn rõ ràng và chính xác (ví dụ: Theo bài báo [Tên bài]).
- Nếu CONTEXT hoàn toàn không có thông tin liên quan, hãy trả lời: "Dựa trên các tài liệu được cung cấp, không có đủ thông tin để trả lời chính xác về..."
"""
```

**3. Kiến trúc Đa Model (Multi-LLM)**

Hệ thống cấu hình hỗ trợ nhiều LLM khác nhau nhằm mục đích dự phòng (fallback) hoặc so sánh chất lượng:
- **Model 1 (Chính)**: Groq API với `qwen-2.5-32b` hoặc `llama-3.1-8b-instant` — siêu tốc độ.
- **Model 2 (Dự phòng)**: Google Gemini `gemini-2.0-flash-exp` — xử lý context dài tốt.

**4. Đánh giá chất lượng RAG (Ragas)**

Chất lượng của RAG pipeline được đo lường bằng framework Ragas với các chỉ số:
- **Context Precision**: Tài liệu liên quan nhất có được xếp hạng cao không.
- **Context Recall**: Hệ thống có lấy đủ thông tin để trả lời không.
- **Faithfulness**: LLM có trung thực với context không (hay tự bịa).
- **Answer Relevancy**: Câu trả lời có đúng trọng tâm câu hỏi không.
