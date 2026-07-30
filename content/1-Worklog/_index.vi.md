---
title: "Nhật ký công việc"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

Trong suốt **8 tuần** thực tập tại chương trình **First Cloud AI Journey** (từ 01/06/2026 đến 14/08/2026), tôi đã tham gia xây dựng dự án **NewsRAG** — hệ thống tổng hợp và phân tích tin tức thông minh dựa trên kiến trúc RAG (Retrieval-Augmented Generation) triển khai hoàn toàn trên AWS.

Vai trò chính của tôi trong nhóm là phụ trách module **Crawl + Queue**, bao gồm: viết Scrapy Spider crawl tin tức từ 3 báo điện tử Việt Nam, đóng gói Docker container cho Fargate Crawler, cấu hình SQS + Dead Letter Queue, và triển khai lên ECR/ECS.

Dưới đây là nhật ký công việc chi tiết theo từng tuần:

**Tuần 1:** [Làm quen AWS & Tìm hiểu dự án NewsRAG](1.1-week1/)

**Tuần 2:** [Nghiên cứu Scrapy Framework & Docker](1.2-week2/)

**Tuần 3:** [Phát triển Scrapy Spider cho NewsRAG](1.3-week3/)

**Tuần 4:** [Dockerize Crawler & Tích hợp ECS Fargate](1.4-week4/)

**Tuần 5:** [Cấu hình SQS & Tích hợp Pipeline](1.5-week5/)

**Tuần 6:** [Terraform Infrastructure & EventBridge Schedule](1.6-week6/)

**Tuần 7:** [Tích hợp hệ thống & Kiểm thử toàn diện](1.7-week7/)

**Tuần 8:** [Hoàn thiện, tối ưu & Viết báo cáo](1.8-week8/)