---
title: "Blog 3: Xây dựng RAG tối ưu"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# Hiện thực hóa hệ thống RAG với Amazon Bedrock và Aurora pgvector

**Retrieval-Augmented Generation (RAG)** là phương pháp kết hợp sức mạnh của Mô hình ngôn ngữ lớn (LLM) với dữ liệu riêng của doanh nghiệp. Để RAG hoạt động hiệu quả, trái tim của hệ thống chính là **Embedding Model** (để biến văn bản thành vector) và **Vector Database** (để lưu trữ và tìm kiếm vector). 

Dưới đây là cách chúng tôi xây dựng trái tim này hoàn toàn bằng hệ sinh thái AWS.

### 1. Amazon Bedrock: Sức mạnh nhúng (Embedding)

Thay vì phải tự host các model như HuggingFace SentenceTransformer trên EC2 (rất tốn kém GPU) hoặc Lambda (chạy chậm và dễ cold-start), chúng tôi sử dụng **Amazon Bedrock** với mô hình `amazon.titan-embed-text-v2:0`.

*   **Hiệu suất và chi phí**: API của Bedrock phản hồi rất nhanh, xử lý embedding cho các đoạn chunk văn bản trong tích tắc. Chi phí được tính theo số token, vô cùng rẻ cho các dự án vừa và nhỏ.
*   **Tính nhất quán tuyệt đối**: Nguyên tắc sống còn của RAG là dữ liệu trong Database và câu hỏi của người dùng (Query) phải được chuyển thành vector bởi **cùng một model**. Bằng cách gọi Bedrock API ở cả khâu ETL và khâu RAG API, chúng tôi đảm bảo tính nhất quán tuyệt đối của không gian vector (vector space).

### 2. Amazon Aurora Serverless v2 + pgvector

Thay vì dùng các Vector Database bên thứ ba như Qdrant, Pinecone hay Milvus, chúng tôi quyết định sử dụng **Amazon Aurora PostgreSQL** kết hợp với extension **`pgvector`**.

*   **Tất cả trong một**: PostgreSQL cho phép lưu trữ siêu dữ liệu (metadata như tác giả, ngày đăng, URL) theo cấu trúc quan hệ (Star Schema) kết hợp với cột kiểu `vector` để lưu embedding. Điều này giúp chúng tôi dễ dàng thực hiện các truy vấn kết hợp: "Tìm các bài báo về kinh tế (vector search) nhưng chỉ lấy những bài xuất bản trong tuần này (SQL filter)".
*   **Chỉ mục HNSW**: Để tìm kiếm nhanh trên hàng trăm ngàn chunk dữ liệu, chúng tôi cấu hình chỉ mục **HNSW (Hierarchical Navigable Small World)** trên cột vector. Nhờ đó, truy vấn tìm kiếm độ tương tự (cosine similarity) trả về kết quả gần như ngay lập tức.
*   **Serverless v2**: Database Aurora tự động scale lên xuống (từ 0.5 đến 2 ACU) dựa trên tải hệ thống, giúp tối ưu chi phí mà vẫn đảm bảo hiệu suất khi có người dùng truy vấn hỏi đáp.

### Kết luận

Kiến trúc kết hợp giữa Amazon Bedrock và Aurora pgvector mang lại một giải pháp RAG khép kín (End-to-End) hoàn toàn trên AWS. Dữ liệu tin tức không bao giờ phải rời khỏi VPC của hệ thống để gọi ra bên ngoài, đảm bảo tính bảo mật và giảm độ trễ mạng tối đa. Đây chính là điểm nhấn kỹ thuật mà chúng tôi tâm đắc nhất trong toàn bộ quy trình xây dựng ứng dụng.