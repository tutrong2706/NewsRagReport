---
title: "Các bài blogs đã đăng"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

Trong quá trình thực tập, tôi đã chuẩn bị 3 bài blog để đăng trên [AWS Study Group](https://www.facebook.com/groups/awsstudygroupfcj), chia sẻ kiến thức và kinh nghiệm từ dự án NewsRAG:

###  [Blog 1 - XÂY DỰNG HỆ THỐNG CRAWL TIN TỨC TỰ ĐỘNG VỚI SCRAPY TRÊN AWS ECS FARGATE](3.1-Blog1/)
Blog chia sẻ kinh nghiệm xây dựng hệ thống crawl tin tức tự động từ 3 báo điện tử Việt Nam (VnExpress, Thanh Niên, VietnamNet) sử dụng Scrapy framework, đóng gói Docker container và triển khai trên AWS ECS Fargate. Bao gồm các vấn đề thực tế gặp phải khi extract thông tin bài viết tiếng Việt.

###  [Blog 2 - SO SÁNH KAFKA VÀ AMAZON SQS: KHI NÀO NÊN DÙNG CÁI NÀO?](3.2-Blog2/)
Blog phân tích chi tiết sự khác biệt giữa Apache Kafka và Amazon SQS, dựa trên kinh nghiệm thực tế khi chuyển đổi từ Kafka sang SQS trong dự án NewsRAG. Giúp người đọc hiểu khi nào nên sử dụng message queue đơn giản thay vì event streaming platform.

###  [Blog 3 - TRIỂN KHAI INFRASTRUCTURE AS CODE VỚI TERRAFORM CHO DỰ ÁN AWS](3.3-Blog3/)
Blog hướng dẫn sử dụng Terraform để quản lý toàn bộ infrastructure AWS (VPC, RDS, ECS, EventBridge) cho một dự án thực tế. Chia sẻ best practices về cách tổ chức Terraform config, quản lý sensitive variables, và tự động hóa deployment.