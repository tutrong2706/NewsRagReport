---
title: "Worklog Tuần 5"
date: 2026-07-28
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu tuần 5:

* Cấu hình Amazon SQS Standard Queue thay thế Kafka (giảm chi phí, đơn giản hóa).
* Thiết lập Dead Letter Queue (DLQ) cho xử lý lỗi.
* Tích hợp luồng Crawler → SQS → Lambda Consumer → PostgreSQL.
* Test end-to-end luồng crawl + consumer.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Tạo SQS Standard Queue: `newsrag-articles` <br> - Cấu hình message retention: 4 ngày <br> - Tạo Dead Letter Queue: `newsrag-articles-dlq` <br> - Cấu hình redrive policy: maxReceiveCount=3 | 14/07/2025   | 14/07/2025      | <https://docs.aws.amazon.com/sqs/>        |
| 3   | - Chuyển đổi `KafkaPipeline` sang `SQSPipeline` (v2): <br>&emsp; + Sử dụng `boto3.client('sqs')` <br>&emsp; + `send_message()` thay vì Kafka produce <br> - So sánh SQS vs Kafka cho use case này | 15/07/2025   | 15/07/2025      |                                           |
| 4   | - Tìm hiểu Lambda Consumer (module của thành viên B): <br>&emsp; + Trigger từ SQS event <br>&emsp; + SHA256 hash URL để deduplicate <br>&emsp; + Insert vào `article_metadata` table <br>&emsp; + ON CONFLICT DO NOTHING | 16/07/2025   | 16/07/2025      | consumer.py                               |
| 5   | - Tích hợp end-to-end: Crawler → SQS → Consumer → PostgreSQL <br> - Test luồng: crawl 100 bài → kiểm tra SQS messages → verify database <br> - Debug: message format, encoding, date parsing | 17/07/2025   | 17/07/2025      |                                           |
| 6   | - Tìm hiểu ETL pipeline (module của thành viên C): <br>&emsp; + `clean_text()`: loại bỏ HTML tags, junk patterns <br>&emsp; + `RecursiveCharacterTextSplitter`: chunk 800 chars, overlap 150 <br>&emsp; + Star Schema: `dim_source`, `dim_time`, `dim_author`, `dim_content`, `fact_articles`, `fact_chunks` | 18/07/2025   | 18/07/2025      | etl_warehouse.py, warehouse.sql           |


### Kết quả đạt được tuần 5:

* Cấu hình SQS thành công:
  * **SQS Standard Queue**: `newsrag-articles` — chi phí ~$0/tháng (so với Kafka tốn thêm quản lý)
  * **Dead Letter Queue**: `newsrag-articles-dlq` — bắt messages bị lỗi sau 3 lần retry
  * Message retention: 4 ngày, visibility timeout: 30 giây

* Hiểu rõ lý do chuyển từ Kafka sang SQS:
  | Tiêu chí        | Kafka (v1)           | SQS (v2)               |
  |-----------------|---------------------|------------------------|
  | Chi phí         | Tốn container Kafka  | ~$0/tháng              |
  | Quản lý         | Phải maintain broker | Fully managed          |
  | Use case        | Real-time streaming  | Batch crawl 1 lần/ngày |
  | Kết luận        | Overkill             | **Phù hợp**           |

* Tích hợp thành công luồng end-to-end:
  * Crawler crawl bài → serialize JSON → push SQS
  * Lambda Consumer trigger từ SQS → parse JSON → SHA256 hash URL → INSERT article_metadata
  * Deduplication hoạt động: `ON CONFLICT (url_hash) DO NOTHING`
  * Test 100 bài: 95 bài insert thành công, 5 bài trùng lặp bị bỏ qua

* Hiểu được Star Schema warehouse:
  * **Dimension tables**: `dim_source` (nguồn báo), `dim_time` (ngày tháng), `dim_author` (tác giả), `dim_content` (nội dung gốc)
  * **Fact tables**: `fact_articles` (bài viết), `fact_chunks` (chunks sau khi cắt), `fact_article_authors` (M:N)
  * Indexes: HNSW cho vector search, B-tree cho url_hash, domain, date
