---
title: "Worklog Tuần 8"
date: 2026-07-28
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---

### Mục tiêu tuần 8:

* Hoàn thiện code, clean up và tối ưu.
* So sánh chi tiết kiến trúc v1 vs v2 (chi phí giảm ~30%).
* Viết tài liệu kỹ thuật `Pipeline_v3.md`.
* Hoàn thiện báo cáo thực tập.

### Các công việc cần triển khai trong tuần này:
| Thứ | Công việc                                                                                                                                                                              | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                            |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2   | - Code review và clean up: <br>&emsp; + Xóa code thừa, comments không cần thiết <br>&emsp; + Thống nhất coding style (PEP 8) <br>&emsp; + Kiểm tra `.env.example` đầy đủ biến         | 21/07/2026   | 21/07/2026      |                                           |
| 3   | - So sánh chi tiết v1 vs v2: <br>&emsp; + Crawler: Lambda (15 phút timeout) → Fargate (không giới hạn) <br>&emsp; + Stream: Kafka → SQS (~$0) <br>&emsp; + Embedding: bge-m3 local → Bedrock API <br>&emsp; + Vector DB: Qdrant Cloud → Aurora pgvector <br>&emsp; + Chi phí: ~$35/tháng → ~$21-26/tháng (giảm ~30%) | 22/07/2026   | 22/07/2026      | Pipeline_v3.md                            |
| 4   | - Viết tài liệu kỹ thuật `Pipeline_v3.md`: <br>&emsp; + Kiến trúc tổng thể (text diagram) <br>&emsp; + Bảng so sánh v1 vs v2 <br>&emsp; + Component map <br>&emsp; + Chi tiết kỹ thuật từng module <br>&emsp; + Chi phí vận hành <br>&emsp; + Kết quả đánh giá Ragas | 23/07/2026   | 23/07/2026      |                                           |
| 5   | - Viết phần phân chia công việc & timeline trong Pipeline_v3.md <br> - Viết hướng phát triển tương lai <br> - Viết hướng dẫn deploy và phát triển local                                  | 24/07/2026   | 24/07/2026      |                                           |
| 6   | - Hoàn thiện báo cáo thực tập (Hugo report) <br> - Kiểm tra lại toàn bộ nội dung worklog 8 tuần <br> - Chuẩn bị slide trình bày (nếu cần) <br> - Nộp báo cáo                           | 25/07/2026   | 25/07/2026      |                                           |


### Kết quả đạt được tuần 8:

* Hoàn thiện code chất lượng sản xuất:
  * Tất cả module đều có error handling, logging
  * `.env.example` đầy đủ 50 biến môi trường
  * `Makefile` với 12 commands tiện lợi (setup, up, down, full, auto, crawl, etl,...)
  * `deploy.sh` cho automated deployment

* Bảng so sánh kiến trúc v1 vs v2:

  | Component       | v1 (Khóa luận)                    | v2 (Cải tiến)                     | Lý do                                |
  |----------------|----------------------------------|----------------------------------|--------------------------------------|
  | Crawler         | Lambda + CrawlSpider (15 phút)   | **Fargate + SitemapSpider**      | Không bị timeout, crawl bài cũ       |
  | Stream          | Kafka trên Docker                | **SQS Standard (~$0)**           | Overkill cho batch pipeline          |
  | Embedding       | BAAI/bge-m3 (1024d, local)       | **Bedrock Titan Embed v2**       | Serverless, không tải model          |
  | Vector DB       | Qdrant Cloud (bên thứ ba)        | **Aurora pgvector**              | Tận dụng RDS, không phụ thuộc ngoài  |
  | **Tổng chi phí**| **~$35/tháng**                   | **~$21-26/tháng**                | **Giảm ~30%**                        |

* Hoàn thành tài liệu `Pipeline_v3.md` (460 dòng) bao gồm:
  * Kiến trúc tổng thể với text diagram
  * Chi tiết 5 module kỹ thuật với code samples
  * Schema SQL (Star Schema + pgvector)
  * Chi phí vận hành chi tiết
  * Kết quả đánh giá Ragas và so sánh với FlashRAG
  * Hướng phát triển tương lai

* Hoàn thiện báo cáo thực tập:
  * 8 tuần worklog chi tiết
  * Proposal dự án NewsRAG
  * Workshop thực tế
  * Tự đánh giá và phản hồi
