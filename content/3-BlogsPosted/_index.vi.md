---
title: "Các bài Blogs & Kinh nghiệm"
date: 2026-07-28
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

{{% notice info %}}
**Ghi chú dành cho Ban Giám Khảo:** 
Theo yêu cầu ban đầu của chương trình thực tập, các bài viết này dự kiến sẽ được đăng tải công khai lên group cộng đồng Facebook. Tuy nhiên, do tính chất đặc thù của dự án (cào và xử lý dữ liệu từ các trang báo điện tử lớn) và để đảm bảo tính an toàn, bảo mật thông tin trước khi dự án hoàn thiện 100%, nhóm chúng tôi đã quyết định **tạm thời chưa đăng tải công khai (publish) lên Facebook**.

Thay vào đó, chúng tôi tổng hợp lại dưới dạng các bài Blog nội bộ trong báo cáo này. Đây chính là **những đúc kết kỹ thuật quý báu và kinh nghiệm thực tế mà cả nhóm đã cùng nhau học được** trong suốt quá trình xây dựng và tối ưu hóa hệ thống News RAG Pipeline.
{{% /notice %}}

Dưới đây là 3 bài blog chia sẻ kinh nghiệm từ dự án NewsRAG:

###  [Blog 1 - VƯỢT GIỚI HẠN TIMEOUT CỦA AWS LAMBDA BẰNG ECS FARGATE](3.1-Blog1/)
Bài viết kể về trải nghiệm thực tế khi chạy hệ thống Crawler. Việc gặp phải giới hạn 15 phút của Lambda đã thôi thúc nhóm phải chuyển đổi linh hoạt sang kiến trúc ECS Fargate, giúp crawler hoạt động ổn định và kéo dài thời gian xử lý an toàn.

###  [Blog 2 - TỐI ƯU HÓA CHI PHÍ HỆ THỐNG: TỪ APACHE KAFKA SANG AMAZON SQS](3.2-Blog2/)
Bài viết phân tích bài toán chi phí khi triển khai trên AWS. Nhóm đã quyết định loại bỏ Apache Kafka (vốn tốn kém chi phí duy trì server 24/7) để chuyển sang dùng Amazon SQS (Serverless), giúp chi phí cho thành phần hàng đợi giảm xuống gần như 0$.

###  [Blog 3 - HIỆN THỰC HÓA HỆ THỐNG RAG KHÉP KÍN VỚI AMAZON BEDROCK VÀ AURORA PGVECTOR](3.3-Blog3/)
Bài viết nhấn mạnh vào tư duy bảo mật và tính đồng nhất kiến trúc. Thay vì sử dụng cơ sở dữ liệu Vector bên thứ ba, nhóm đã tích hợp thành công Amazon Bedrock và Aurora Serverless v2 + pgvector để toàn bộ luồng RAG được xử lý hoàn toàn khép kín bên trong môi trường AWS.