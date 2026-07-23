---
title: "Event 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 4.2. </b> "
---

# Bài thu hoạch “AWS Serverless & Container Day”

### Mục Đích Của Sự Kiện

- Giới thiệu các xu hướng mới nhất về công nghệ Serverless và Container trên AWS.
- Phân tích sâu về Amazon ECS, EKS và AWS Fargate.
- Hướng dẫn xây dựng kiến trúc microservices tối ưu chi phí và hiệu suất.
- Chia sẻ kinh nghiệm thực tiễn từ các chuyên gia AWS và khách hàng.

### Nội Dung Nổi Bật

#### 1. Tối ưu hóa Container với AWS Fargate
- **Không quản lý server**: Fargate giúp các lập trình viên tập trung vào ứng dụng thay vì phải lo lắng về việc quản lý EC2 instances.
- **Tính năng Right-sizing**: Cấu hình CPU và Memory chính xác cho từng task để tiết kiệm chi phí.
- **Tích hợp với Graviton**: Chạy Fargate trên nền tảng chip ARM (Graviton) giúp tăng hiệu năng lên 40% với chi phí thấp hơn 20%.

#### 2. Event-Driven Architecture với EventBridge
- Sử dụng Amazon EventBridge để lên lịch (schedule) hoặc phản hồi các sự kiện từ các dịch vụ khác (S3, SQS).
- Cách EventBridge giảm bớt việc viết code tích hợp (integration code) giữa các microservices.

#### 3. Best Practices cho AWS Lambda
- Quản lý **Cold Start**: Sử dụng Provisioned Concurrency.
- Lựa chọn kích thước memory phù hợp để đạt sự cân bằng hoàn hảo giữa hiệu suất và chi phí.
- Giới hạn thời gian (Timeout): Lưu ý giới hạn 15 phút của Lambda đối với các tác vụ dài.

### Ứng Dụng Vào Công Việc (Dự án NewsRAG)

- **Chuyển đổi kiến trúc Crawler**: Kiến thức về giới hạn 15 phút của Lambda đã giúp tôi nhận ra kiến trúc v1 của NewsRAG (dùng Lambda để crawl web) không khả thi cho SitemapSpider. Từ sự kiện này, tôi đã đề xuất và trực tiếp chuyển đổi Crawler sang chạy trên container sử dụng **AWS Fargate**, giải quyết triệt để vấn đề timeout.
- **Tự động hóa pipeline**: Ứng dụng **Amazon EventBridge Scheduler** để tự động kích hoạt Fargate Crawler mỗi ngày vào lúc 01:00 UTC thay vì phải chạy thủ công.
- **Tối ưu chi phí Docker**: Dựa trên best practices từ sự kiện, tôi đã tối ưu hóa Dockerfile cho Crawler (sử dụng `python:3.10-slim`) và thiết lập cấu hình Fargate ở mức nhỏ nhất (0.25 vCPU, 512MB RAM) vì tác vụ I/O bound không cần nhiều compute.

### Trải nghiệm trong event

Sự kiện **"AWS Serverless & Container Day"** diễn ra vào đúng thời điểm nhóm NewsRAG đang gặp bế tắc với lỗi timeout của Lambda Crawler. Những kiến thức thu được giống như một chiếc "phao cứu sinh".

#### Sự tương tác & Hỏi đáp
- Tôi đã có cơ hội đặt câu hỏi trực tiếp cho các Solutions Architect của AWS về bài toán web scraping dài hạn và nhận được lời khuyên rõ ràng: *"Sử dụng ECS Fargate thay vì Lambda cho các tác vụ không thể đoán trước thời gian hoàn thành"*.

#### Giao lưu kỹ thuật
- Gặp gỡ nhiều lập trình viên khác và nghe họ chia sẻ về những sai lầm khi cấu hình container, giúp tôi tránh được những cạm bẫy tương tự khi viết file `main.tf` Terraform cho dự án của mình.

#### Bài học tâm đắc
- Công nghệ nào cũng có use case riêng của nó. Lambda rất tuyệt vời cho API (như RAG API của dự án), nhưng Fargate mới là "chân ái" cho những batch job kéo dài như Crawler và ETL.

#### Hình ảnh sự kiện
*Thêm hình ảnh thực tế của bạn tại đây*

> Sự kiện là một bước ngoặt lớn trong quá trình thiết kế hệ thống NewsRAG v2, mang lại sự ổn định và hiệu năng vượt trội cho toàn bộ pipeline dữ liệu.
