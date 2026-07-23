---
title: "Worklog Tuần 7"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Mục tiêu tuần 7:

* Integration test toàn bộ pipeline: Crawl → ETL → Vectorize → RAG API.
* Debug và xử lý edge cases trong quá trình crawl (author, date, encoding).
* Tìm hiểu hệ thống đánh giá Ragas và so sánh NewsRAG với FlashRAG.
* Tối ưu hiệu suất crawl và xử lý lỗi.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Integration test: chạy full pipeline `python main.py --mode full` <br> - Kiểm tra luồng: Spider → Kafka → Consumer → PostgreSQL → ETL → Vectorize <br> - Monitor logs từ CloudWatch    | 28/07/2025   | 28/07/2025      |                                           |
| 3   | - Debug edge cases trong spider: <br>&emsp; + Author chứa URL, ngày tháng, ký tự đặc biệt <br>&emsp; + Bài viết không có title hoặc content < 100 chars <br>&emsp; + Date format không chuẩn từ các báo khác nhau <br>&emsp; + Encoding issues với tiếng Việt (UTF-8) | 29/07/2025   | 29/07/2025      |                                           |
| 4   | - Cải thiện author validation trong spider: <br>&emsp; + Thêm `fake_authors` list (vietnamnet news, ban bien tap,...) <br>&emsp; + Kiểm tra `is_valid_author()`: loại bỏ tên chứa bad_words (thứ hai, ngày, tháng,...) <br>&emsp; + Thêm bottom paragraph scanning cho pattern "Theo: ..." | 30/07/2025   | 30/07/2025      |                                           |
| 5   | - Tìm hiểu Ragas evaluation framework: <br>&emsp; + Faithfulness: mức trung thực so với ngữ cảnh <br>&emsp; + Answer Relevancy: liên quan với câu hỏi <br>&emsp; + Context Precision: xếp hạng tài liệu <br>&emsp; + Context Recall: lấy đủ tài liệu <br> - So sánh kết quả NewsRAG vs FlashRAG | 31/07/2025   | 31/07/2025      | Pipeline_v3.md                            |
| 6   | - Tối ưu Crawler: <br>&emsp; + Điều chỉnh CONCURRENT_REQUESTS, DOWNLOAD_DELAY theo OS <br>&emsp; + Cải thiện xử lý retry khi request fail <br>&emsp; + Thêm ROBOTSTXT_OBEY config <br> - Test lại full pipeline sau tối ưu | 01/08/2025   | 01/08/2025      |                                           |


### Kết quả đạt được tuần 7:

* Integration test thành công trên full pipeline:
  * Crawl ~500 bài/lần chạy từ 3 báo
  * ETL xử lý: clean HTML → chunk 800 chars (overlap 150) → insert Star Schema
  * Vectorize: embedding bằng `BAAI/bge-small-en-v1.5` → upsert vào Qdrant
  * RAG API: query → embed → search top-k → LLM generate answer

* Xử lý edge cases quan trọng:
  * **Author invalid**: Phát hiện và loại bỏ 15+ loại fake author (URL, ngày tháng, tên báo, ký tự đặc biệt)
  * **Date parsing**: Hỗ trợ 3 format (ISO, VN dd/mm/yyyy, date-only) + fallback qua 10 CSS selectors
  * **Empty content**: Skip bài viết có content < 100 ký tự
  * **ETL author cleanup**: Loại bỏ author dài > 40 chars, chứa http/|/@

* Kết quả đánh giá Ragas:
  | Chỉ số                | NewsRAG        | FlashRAG       |
  |----------------------|----------------|----------------|
  | **Context Precision** | **Cao hơn**    | Trung bình     |
  | **Context Recall**    | Trung bình     | **Cao hơn**    |
  | **Faithfulness**      | Trung bình     | **Cao hơn**    |
  | **Answer Relevancy**  | Tương đương    | Tương đương    |

  > Tổng thể các chỉ số ở mức khiêm tốn (dưới 0.5) do prompt RAG được cấu hình trả lời chi tiết, diễn giải đầy đủ → làm giảm cosine similarity khi Ragas dịch ngược thành câu hỏi.

* Tối ưu hiệu suất Crawler:
  * Windows: CONCURRENT_REQUESTS=16, DOWNLOAD_DELAY=1.0
  * Linux (Fargate): CONCURRENT_REQUESTS=32, DOWNLOAD_DELAY=0.5
  * User-Agent tùy chỉnh theo OS
