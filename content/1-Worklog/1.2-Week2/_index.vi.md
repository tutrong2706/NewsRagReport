---
title: "Worklog Tuần 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu tuần 2:

* Nghiên cứu sâu về Scrapy framework — công cụ chính để crawl tin tức.
* Tìm hiểu sự khác biệt giữa CrawlSpider (v1) và SitemapSpider (v2).
* Nắm vững Docker, Dockerfile, docker-compose để chuẩn bị containerize.
* Phân tích cấu trúc sitemap XML của 3 báo điện tử Việt Nam.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                        | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Tìm hiểu Scrapy framework: <br>&emsp; + Cấu trúc project Scrapy <br>&emsp; + Spider, Pipeline, Settings <br>&emsp; + Middleware và Item Pipeline                                | 23/06/2025   | 23/06/2025      | <https://docs.scrapy.org/>                |
| 3   | - So sánh CrawlSpider vs SitemapSpider: <br>&emsp; + CrawlSpider: dựa vào DEPTH_LIMIT, bỏ sót bài cũ <br>&emsp; + SitemapSpider: đọc toàn bộ URL từ sitemap, không phụ thuộc navigation | 24/06/2025   | 24/06/2025      | <https://docs.scrapy.org/>                |
| 4   | - Phân tích cấu trúc sitemap XML của 3 báo: <br>&emsp; + VnExpress: `sitemap_news.xml` <br>&emsp; + Thanh Niên: `sitemap.xml` <br>&emsp; + VietnamNet: `sitemap_news.xml`          | 25/06/2025   | 25/06/2025      |                                           |
| 5   | - Tìm hiểu Docker cơ bản: <br>&emsp; + Dockerfile: FROM, WORKDIR, COPY, RUN, CMD <br>&emsp; + docker-compose.yml: services, volumes, environment <br>&emsp; + Multi-stage build    | 26/06/2025   | 26/06/2025      | <https://docs.docker.com/>                |
| 6   | - Tìm hiểu thư viện `newspaper3k` để extract article content <br> - Nghiên cứu Kafka Producer/Consumer pattern <br> - Tìm hiểu `confluent_kafka` Python client                    | 27/06/2025   | 27/06/2025      | <https://newspaper.readthedocs.io/>       |


### Kết quả đạt được tuần 2:

* Nắm vững cấu trúc project Scrapy:
  * `spiders/`: chứa các spider class
  * `pipelines.py`: xử lý item sau khi spider yield
  * `settings.py`: cấu hình CONCURRENT_REQUESTS, DOWNLOAD_DELAY, DEPTH_LIMIT,...

* Hiểu rõ sự khác biệt giữa 2 loại Spider:
  * **CrawlSpider (v1)**: Dùng `DEPTH_LIMIT=5`, chỉ crawl được trang trong phạm vi liên kết → bỏ sót bài cũ không được link từ trang chủ
  * **SitemapSpider (v2)**: Đọc trực tiếp từ file sitemap XML → crawl toàn bộ bài viết bao gồm cả bài cũ, không bị timeout 15 phút như Lambda

* Đã phân tích sitemap XML của 3 báo:
  * VnExpress: `https://vnexpress.net/sitemap_news.xml`
  * Thanh Niên: `https://thanhnien.vn/sitemap.xml`
  * VietnamNet: `https://vietnamnet.vn/sitemap_news.xml`

* Nắm được kiến thức Docker cần thiết:
  * Cách viết Dockerfile tối ưu (base image `python:3.10-slim`, multi-layer caching)
  * Docker Compose để chạy PostgreSQL + Kafka local
  * Hiểu biến môi trường `.env` và cách inject vào container

* Tìm hiểu `newspaper3k`:
  * Thư viện Python hỗ trợ extract title, content, author, publish_date từ HTML
  * Dùng `article.set_html()` + `article.parse()` thay vì download trực tiếp
  * Phối hợp với CSS Selectors của Scrapy để tăng độ chính xác
