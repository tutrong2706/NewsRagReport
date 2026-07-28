---
title: "Blog 1: Vượt giới hạn Timeout với AWS Fargate"
date: 2026-07-28
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Vượt qua giới hạn 15 phút của AWS Lambda bằng kiến trúc ECS Fargate

Trong quá trình xây dựng hệ thống **News RAG Pipeline**, một trong những thách thức kỹ thuật lớn nhất mà nhóm gặp phải là việc cào dữ liệu (crawl) tin tức từ các trang báo điện tử lớn như VnExpress, Thanh Niên và VietnamNet. Ban đầu, hệ thống được thiết kế chạy trên **AWS Lambda**.

### Bài toán đặt ra: Giới hạn thời gian (Timeout)

AWS Lambda là một dịch vụ tuyệt vời để chạy mã nguồn mà không cần quản lý máy chủ (serverless). Tuy nhiên, nó có một **giới hạn cứng là 15 phút (900 giây)** cho mỗi lần thực thi.

Khi crawler của chúng tôi quét qua các `sitemap` của các trang báo, số lượng bài viết cần xử lý thường lên tới hàng ngàn bài mỗi ngày. Việc truy cập, tải nội dung HTML, bóc tách dữ liệu (parsing) và đẩy vào hàng đợi mất rất nhiều thời gian do phải tôn trọng tốc độ tải trang và quy định của máy chủ nguồn (tránh bị block IP). Do đó, Lambda function thường xuyên bị **timeout** trước khi hoàn thành công việc.

### Giải pháp: Chuyển đổi sang Amazon ECS Fargate

Để giải quyết bài toán này mà vẫn giữ được tinh thần "Serverless" (không phải duy trì máy chủ chạy 24/7), chúng tôi quyết định chuyển đổi kiến trúc sang **Amazon ECS Fargate**.

**Lý do chọn Fargate:**
1. **Không giới hạn thời gian thực thi**: Task trên Fargate có thể chạy bao lâu tùy ý cho đến khi hoàn thành công việc.
2. **Khởi chạy theo lịch trình (Scheduled Tasks)**: Kết hợp với **Amazon EventBridge**, chúng tôi dễ dàng lên lịch cho Crawler chạy tự động vào lúc 01:00 và 02:00 sáng mỗi ngày.
3. **Mở rộng dễ dàng**: Khi cần cào thêm nhiều trang báo khác, chỉ cần tăng số lượng CPU và RAM cấp phát cho Task (hiện tại chỉ dùng 0.25 vCPU và 0.5 GB RAM là đủ).
4. **Đóng gói Docker**: Scrapy framework và các thư viện phụ thuộc được đóng gói gọn gàng trong một Docker Image và lưu trữ trên **Amazon ECR**.

### Kết quả đạt được

Việc chuyển đổi sang kiến trúc ECS Fargate đã mang lại sự ổn định tuyệt đối cho luồng thu thập dữ liệu. Crawler hiện tại có thể chạy liên tục trong 30-40 phút mỗi đêm để lấy về hàng trăm bài báo mới mà không hề gặp lỗi gián đoạn. Hơn nữa, chi phí cho Fargate lại cực kỳ tối ưu vì chúng tôi chỉ trả tiền cho chính xác số phút tính toán mà Crawler thực sự chạy.

*Đúc kết kinh nghiệm: Không có một dịch vụ AWS nào là "chìa khóa vạn năng". Việc hiểu rõ giới hạn (limits) của từng dịch vụ như Lambda và linh hoạt chuyển đổi sang các dịch vụ phù hợp hơn như ECS Fargate là một kỹ năng quan trọng trong thiết kế kiến trúc Đám mây.*