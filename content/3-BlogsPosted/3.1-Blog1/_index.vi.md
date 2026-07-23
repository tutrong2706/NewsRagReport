---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Xây dựng Hệ thống Crawl Tin tức Tự động với Scrapy trên AWS ECS Fargate

Trong dự án NewsRAG, nhóm cần một hệ thống crawl tin tức tự động, ổn định, chạy hàng ngày để thu thập dữ liệu từ 3 báo điện tử lớn: **VnExpress**, **Thanh Niên**, **VietnamNet**. Bài viết này chia sẻ kinh nghiệm xây dựng hệ thống crawler sử dụng Scrapy framework, đóng gói Docker và triển khai trên AWS ECS Fargate.

### Tại sao chọn Scrapy?

- **Mạnh mẽ và linh hoạt**: Hỗ trợ Spider, Pipeline, Middleware — dễ mở rộng
- **Xử lý đồng thời**: CONCURRENT_REQUESTS có thể điều chỉnh (16–32 requests cùng lúc)
- **Hệ sinh thái phong phú**: SitemapSpider, CrawlSpider, nhiều extension
- **Tương thích tốt với newspaper3k**: Thư viện extract content tiếng Việt

### Thách thức khi crawl tin tức tiếng Việt

1. **Author extraction**: Mỗi báo có cấu trúc HTML khác nhau. Phải xây dựng hệ thống fallback qua 12+ CSS selectors
2. **Date parsing**: Hỗ trợ cả ISO format và VN format (dd/mm/yyyy HH:MM)
3. **Fake authors**: Nhiều trường hợp author là tên báo, nhãn chung chung → cần hàm `is_valid_author()` lọc
4. **Encoding**: Đảm bảo UTF-8 cho tiếng Việt trong toàn bộ pipeline

### Kiến trúc Crawler

```
Scrapy Spider → KafkaPipeline → Kafka/SQS → Consumer → PostgreSQL
     ↓
parse_article()
     ↓
newspaper3k + CSS Selectors
     ↓
{title, content, author, publish_date, url, source}
```

### Triển khai trên ECS Fargate

- **Dockerfile**: Base `python:3.10-slim`, cài `libpq-dev` cho psycopg2
- **ECS Task Definition**: 0.25 vCPU, 512 MB RAM (đủ cho crawl)
- **EventBridge**: Chạy tự động lúc 01:00 UTC hàng ngày
- **CloudWatch Logs**: Monitor real-time

### Kết quả

- Crawl ~500 bài/lần chạy từ 3 nguồn
- Thời gian chạy: ~30 phút
- Chi phí Fargate: ~$3-5/tháng

...Hình ảnh...

...Link bài blog...