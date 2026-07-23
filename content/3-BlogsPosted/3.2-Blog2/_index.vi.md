---
title: "Blog 2"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# So sánh Kafka và Amazon SQS: Khi nào nên dùng cái nào?

Trong dự án NewsRAG, nhóm đã chuyển đổi từ **Apache Kafka** (v1) sang **Amazon SQS** (v2) cho phần message queue. Bài viết này chia sẻ phân tích chi tiết và kinh nghiệm thực tế khi đưa ra quyết định này.

### Bối cảnh

- **v1**: Sử dụng Kafka chạy trên Docker container, cần quản lý broker, ZooKeeper
- **v2**: Chuyển sang SQS Standard — fully managed, chi phí gần như $0

### So sánh chi tiết

| Tiêu chí              | Kafka                              | Amazon SQS                       |
|-----------------------|-------------------------------------|----------------------------------|
| **Kiến trúc**         | Distributed log, multi-broker       | Managed message queue            |
| **Message ordering**  | Có (theo partition)                 | Best-effort (Standard)           |
| **Throughput**        | Rất cao (100K+ msg/s)              | Cao (3000 msg/s per queue)       |
| **Message retention** | Cấu hình (mặc định 7 ngày)        | Tối đa 14 ngày                  |
| **Consumer groups**   | Native support                      | Không có                         |
| **Chi phí**           | Infrastructure + management         | Pay-per-request (~$0 cho <1M)    |
| **Quản lý**           | Self-managed hoặc MSK              | Fully managed                    |
| **Dead Letter Queue** | Phải tự implement                   | Native support                   |

### Khi nào nên dùng Kafka?

- Real-time streaming với throughput cực cao
- Cần event replay (đọc lại message)
- Microservices phức tạp với nhiều consumer groups
- Log aggregation quy mô lớn

### Khi nào nên dùng SQS?

- Batch processing (crawl 1 lần/ngày)
- Pipeline đơn giản (1 producer, 1 consumer)
- Muốn giảm chi phí vận hành
- Cần DLQ sẵn có cho error handling

### Kết luận từ NewsRAG

Với use case crawl tin tức 1 lần/ngày (~500 bài), Kafka là **overkill**. SQS đáp ứng hoàn toàn yêu cầu với chi phí gần $0 và không cần quản lý.

...Hình ảnh...

...Link bài blog...