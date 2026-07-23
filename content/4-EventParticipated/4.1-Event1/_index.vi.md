---
title: "Event 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---

# Bài thu hoạch “GenAI-powered App-DB Modernization workshop”

### Mục Đích Của Sự Kiện

- Chia sẻ best practices trong thiết kế ứng dụng hiện đại
- Giới thiệu phương pháp DDD và event-driven architecture
- Hướng dẫn lựa chọn compute services phù hợp
- Giới thiệu công cụ AI hỗ trợ development lifecycle

### Danh Sách Diễn Giả

- **Jignesh Shah** - Director, Open Source Databases
- **Erica Liu** - Sr. GTM Specialist, AppMod
- **Fabrianne Effendi** - Assc. Specialist SA, Serverless Amazon Web Services

### Nội Dung Nổi Bật

#### Chuyển đổi sang kiến trúc ứng dụng mới - Microservice Architecture
Chuyển đổi thành hệ thống modular – từng chức năng là một **dịch vụ độc lập** giao tiếp với nhau qua **sự kiện** với 3 trụ cột cốt lõi:
- **Queue Management**: Xử lý tác vụ bất đồng bộ (Amazon SQS)
- **Caching Strategy:** Tối ưu performance
- **Message Handling:** Giao tiếp linh hoạt giữa services

#### Event-Driven Architecture
- **3 patterns tích hợp**: Publish/Subscribe, Point-to-point, Streaming
- **Lợi ích**: Loose coupling, scalability, resilience
- **So sánh sync vs async**: Hiểu rõ trade-offs (sự đánh đổi)

#### Compute Evolution
- **Shared Responsibility Model**: Từ EC2 → ECS → Fargate → Lambda
- **Serverless benefits**: Không cần quản lý server, auto-scaling, chỉ trả phí khi sử dụng
- **Functions vs Containers**: Tiêu chí lựa chọn phù hợp cho từng workload

### Ứng Dụng Vào Công Việc (Dự án NewsRAG)

- **Implement event-driven patterns**: Ứng dụng SQS làm queue trung gian giữa Crawler và Consumer trong pipeline NewsRAG, giúp loose coupling hai thành phần này.
- **Serverless adoption**: Sử dụng ECS Fargate cho module Crawler và ETL, và AWS Lambda cho module Consumer và RAG API, tận dụng tối đa kiến trúc serverless để tối ưu chi phí (giảm 30% so với v1).
- **Compute selection**: Áp dụng bài học Functions vs Containers để quyết định chuyển Crawler từ Lambda sang ECS Fargate để tránh lỗi timeout 15 phút.

### Trải nghiệm trong event

Tham gia workshop **“GenAI-powered App-DB Modernization”** là một trải nghiệm rất bổ ích, giúp tôi có cái nhìn toàn diện về cách hiện đại hóa ứng dụng và cơ sở dữ liệu bằng các phương pháp và công cụ hiện đại. Một số trải nghiệm nổi bật:

#### Học hỏi từ các diễn giả có chuyên môn cao
- Các diễn giả đến từ AWS đã chia sẻ **best practices** trong thiết kế ứng dụng hiện đại, đặc biệt là cách tận dụng GenAI để tối ưu hóa quy trình.

#### Trải nghiệm kỹ thuật thực tế
- Hiểu rõ trade-offs giữa **synchronous và asynchronous communication** cũng như các pattern tích hợp, từ đó tự tin áp dụng SQS cho dự án NewsRAG.
- Phân biệt rõ ưu nhược điểm giữa **ECS Fargate và Lambda**, một kiến thức then chốt giúp nhóm tái cấu trúc hệ thống NewsRAG v2.

#### Kết nối và trao đổi
- Workshop tạo cơ hội trao đổi trực tiếp với các chuyên gia và học viên khác trong chương trình FCAJ, giúp giải đáp nhiều thắc mắc về serverless architecture.

#### Bài học rút ra
- Việc áp dụng event-driven patterns giúp giảm **coupling**, tăng **scalability** và **resilience** cho hệ thống.
- Lựa chọn đúng Compute Service (Fargate vs Lambda) là yếu tố sống còn cho sự ổn định và chi phí của hệ thống.

#### Hình ảnh sự kiện
*Thêm hình ảnh thực tế của bạn tại đây*

> Tổng thể, sự kiện không chỉ cung cấp kiến thức kỹ thuật mà còn giúp tôi thay đổi cách tư duy về thiết kế hệ thống, đóng góp trực tiếp vào thành công của dự án NewsRAG.
