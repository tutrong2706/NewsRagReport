---
title: "Blog 2: Tối ưu chi phí bằng SQS"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# Tối ưu hóa chi phí hệ thống: Chuyển đổi từ Apache Kafka sang Amazon SQS

Một tiêu chí quan trọng khi thiết kế hệ thống trên Cloud là **Tối ưu hóa chi phí (Cost Optimization)**. Trong phiên bản đầu tiên (v1) phát triển ở môi trường cục bộ (local), dự án sử dụng **Apache Kafka** làm hệ thống hàng đợi (message broker) để luân chuyển dữ liệu từ Crawler sang Database. Tuy nhiên, khi đưa lên AWS, bài toán chi phí trở thành rào cản lớn.

### Bài toán chi phí với Kafka

Apache Kafka là một công cụ stream processing cực kỳ mạnh mẽ. Nhưng để chạy Kafka trên AWS, chúng ta có hai lựa chọn chính:
1. **Amazon MSK (Managed Streaming for Apache Kafka)**: Chi phí rất đắt đỏ (có thể lên tới hàng chục hoặc hàng trăm USD mỗi tháng).
2. **Tự host Kafka trên EC2**: Phải duy trì máy chủ EC2 chạy 24/7, tốn kém chi phí cố định (ít nhất ~$10-15/tháng) cộng với công sức quản trị hệ thống, cập nhật bảo mật và duy trì ZooKeeper.

Trong khi đó, luồng dữ liệu của NewsRAG có đặc thù là **Batch processing**: Crawler chỉ chạy 1-2 lần mỗi ngày (vào ban đêm) và đẩy khoảng 500 - 1000 tin tức mới. Việc duy trì một cụm Kafka chạy 24/7 chỉ để phục vụ luồng dữ liệu đứt quãng này là vô cùng lãng phí.

### Lựa chọn thay thế: Amazon SQS (Simple Queue Service)

Chúng tôi đã quyết định thay thế toàn bộ kiến trúc hàng đợi sang **Amazon SQS**.

**Lý do SQS là lựa chọn hoàn hảo:**
1. **Fully Serverless & Pay-as-you-go**: SQS không yêu cầu máy chủ. Bạn chỉ trả tiền dựa trên số lượng request. Với lượng tin tức hàng ngày của hệ thống, số lượng request hoàn toàn nằm gọn trong **Free Tier** (1 triệu request đầu tiên mỗi tháng là miễn phí). Chi phí hàng đợi được giảm xuống **còn $0**.
2. **Tích hợp tự nhiên với AWS Lambda**: SQS có thể đóng vai trò là "Event Source" để tự động kích hoạt Lambda (Lambda Consumer). Khi Crawler đẩy bài báo vào SQS, SQS sẽ tự động gọi Lambda để lưu vào Database mà không cần chúng ta phải tự viết code vòng lặp lắng nghe (polling).
3. **Dead-Letter Queue (DLQ)**: SQS cung cấp sẵn cơ chế DLQ. Nếu một bài báo bị lỗi định dạng và Lambda không thể lưu vào DB sau nhiều lần thử, message đó sẽ được đẩy sang DLQ để chúng tôi có thể phân tích sau, đảm bảo không bị mất mát dữ liệu.

### Đánh giá từ quyết định thiết kế

Bằng việc loại bỏ Kafka và chuyển sang SQS, hệ thống không chỉ trở nên nhẹ nhàng, dễ bảo trì hơn mà còn tiết kiệm được một khoản ngân sách vận hành rất lớn. Đây là một minh chứng thực tế cho thấy: **Lựa chọn công nghệ không phải là chọn thứ "xịn nhất", mà là chọn thứ "phù hợp nhất" với quy mô và bài toán hiện tại.**