---
title: "Worklog Tuần 3"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu tuần 3:

* Phát triển hoàn chỉnh `NewsRAGSpider` — spider chính để crawl tin tức.
* Xử lý logic extract thông tin bài viết: title, content, author, publish_date.
* Viết `KafkaPipeline` để đẩy dữ liệu crawl được vào Kafka topic.
* Test crawl thử trên local.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Viết class `NewsRAGSpider` kế thừa `scrapy.Spider` <br> - Implement `parse()` method: lọc URL nội bộ, phân loại link bài viết (.html, .htm) và link danh mục <br> - Cấu hình `custom_settings`: CONCURRENT_REQUESTS, DOWNLOAD_DELAY, DEPTH_LIMIT | 30/06/2025   | 30/06/2025      | spider.py                                 |
| 3   | - Implement `parse_article()`: sử dụng `newspaper3k` để parse HTML <br> - Lọc bài viết có nội dung < 100 ký tự <br> - Extract author từ `article.authors`                               | 01/07/2025   | 01/07/2025      |                                           |
| 4   | - Xử lý author extraction nâng cao: <br>&emsp; + Fallback qua nhiều CSS selectors (`.author-name`, `.tac-gia`, `a[rel="author"]`,...) <br>&emsp; + Validate author: loại bỏ fake authors (URLs, ngày tháng, tên báo) <br>&emsp; + Hàm `is_valid_author()`: kiểm tra độ dài, ký tự đặc biệt, bad words | 02/07/2025   | 02/07/2025      |                                           |
| 5   | - Xử lý publish_date parsing: <br>&emsp; + Parse ISO format: `2025-07-01T14:30` <br>&emsp; + Parse VN format: `01/07/2025 14:30` <br>&emsp; + Fallback qua nhiều CSS selectors + `article.publish_date` <br> - Viết `KafkaPipeline`: serialize item thành JSON → push vào Kafka topic `news_raw` | 03/07/2025   | 03/07/2025      | pipelines.py                              |
| 6   | - Test crawl local: `scrapy crawl news_rag_spider` <br> - Kiểm tra output: đúng format JSON, author hợp lệ, date đúng <br> - Debug và sửa lỗi CSS selectors cho từng báo                | 04/07/2025   | 04/07/2025      |                                           |


### Kết quả đạt được tuần 3:

* Hoàn thành `NewsRAGSpider` (`crawler/spiders/spider.py`) với các tính năng:
  * **parse()**: Duyệt tất cả link trên trang, lọc URL cùng domain, phân loại bài viết và trang danh mục
  * **parse_article()**: Sử dụng `newspaper3k` kết hợp CSS Selectors
  * **Custom settings**: CONCURRENT_REQUESTS=16 (Windows) / 32 (Linux), DOWNLOAD_DELAY=1.0/0.5, DEPTH_LIMIT=5

* Xây dựng hệ thống extract author phức tạp với multi-level fallback:
  1. Lấy từ `newspaper3k` → validate
  2. Fallback qua 12+ CSS selectors: `.author-name`, `.tac-gia`, `a[href*="tac-gia"]`,...
  3. Scan bottom paragraphs của bài viết tìm pattern "Theo: ..." / "Nguon: ..."
  4. Cuối cùng fallback về `meta[name="author"]`
  * Hàm `is_valid_author()` kiểm tra: độ dài 2-100 ký tự, không chứa URL/email, không chứa ngày tháng, không phải bad words (tên báo, nhãn chung chung)

* Xử lý date parsing đa dạng:
  * ISO format: `2025-07-01T14:30:00`
  * VN format: `01/07/2025 14:30` hoặc `01-07-2025`
  * Fallback qua 10 CSS selectors + `article.publish_date`
  * Output chuẩn: `YYYY-MM-DD HH:MM:SS`

* Hoàn thành `KafkaPipeline` (`crawler/pipelines.py`):
  * Serialize item thành JSON (ensure_ascii=False cho tiếng Việt)
  * Push vào Kafka topic `news_raw`
  * Producer flush sau mỗi item

* Test crawl thành công trên local, output format hợp lệ
