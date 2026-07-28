---
title: "Báo cáo Thu hoạch Event 3: Ứng dụng Agentic AI và Trải nghiệm Thực chiến Hackathon"
date: 2026-06-28
weight: 3
chapter: false
pre: " 3. "
---

# Báo cáo Thu hoạch: Ứng dụng Agentic AI và Trải nghiệm Thực chiến Hackathon

## 1. Mục tiêu Sự kiện
- Trải nghiệm hành trình dấn thân, đối mặt với áp lực thời gian và rèn luyện tư duy phát triển sản phẩm tinh gọn (MVP) thông qua sức nóng của cuộc thi Agentic AI Build Week Hackathon.
- Khám phá sức mạnh đột phá của Agentic AI trong việc giải quyết các bài toán vận hành thực tế của doanh nghiệp, trải dài từ kiểm soát đám đông, đặt món ăn đa kênh đến phân tích chiến lược rủi ro và hỗ trợ thiết kế kiến trúc hệ thống đám mây.

## 2. Diễn giả Khách mời (Các Đội thi)
- **Team 3KA** (Huỳnh An Khương, Nguyễn Quốc Huy, Ngô Quang Khôi, Hoàng Lê Thành Đức, Đặng Nguyễn Phước Lộc, Đặng Trường Hưng) - Dự án S.H.E.P.H.E.R.D (Giám sát và dự báo dòng người).
- **One Team** (Anh Duy, Tran Dong, Doan Trung, Minh Viet, Anshul Roy) - Dự án KFC Bot Agent (Trợ lý đặt món đa kênh).
- **Plan V** (Pham Tien Thuan Phat, Huynh Hoang Long, Le Minh Nghia, Tran Dai Vi, Nguyen An) - Dự án SA Professional AI Native App (Tác tử hỗ trợ kiến trúc sư đám mây).
- **Team Signal Scout** (Le Tan Luc, Do Hoang Hieu, Trieu Quoc Hao, Nguyen Van Duy Khiem, Nguyen Cong Minh, Nguyen Tran Minh Quan) - Dự án Signal Scout (Hệ thống phân tích rủi ro doanh nghiệp).

## 3. Những Điểm Nhấn Quan Trọng (Key Highlights)

### Sự bứt phá của Agentic AI: Khi AI biết tự lập kế hoạch và hành động
Agentic AI đánh dấu một bước chuyển dịch mang tính cách mạng khi AI không còn chỉ phản hồi thụ động. Thay vào đó, AI đã có khả năng tự động lập kế hoạch, gọi các công cụ nghiệp vụ (tools/APIs) để giải quyết những bài toán phức tạp và xác minh kết quả một cách độc lập.

Điều này được minh chứng rất rõ nét qua dự án KFC Bot Agent — hệ thống giúp khách hàng đặt món trực tiếp trên Zalo/Messenger mà không cần chuyển app hay đăng ký rườm rà. Hoặc ấn tượng hơn là ứng dụng SA Professional có thể tự động bóc tách yêu cầu nghiệp vụ (BRD), vẽ sơ đồ kiến trúc trên Draw.io và ước tính chi phí hạ tầng AWS chỉ trong vài phút.

### Hành trình 24h Hackathon: Đầy hỗn loạn nhưng vô giá
Cuộc thi Hackathon thực sự là một cuộc "hành xác" ngọt ngào và đầy cảm xúc, đưa các đội thi qua đủ mọi cung bậc: từ lo âu, cạn kiệt năng lượng cho đến khoảnh khắc vỡ òa hân hoan khi sản phẩm thực sự hoạt động.

Những câu chuyện hậu trường "dở khóc dở cười" như: tranh cãi nội bộ vì chưa phân định rõ vai trò, thức trắng đêm debug code đến 3 giờ sáng, uống cạn 5 lon bò húc, quên commit code hay vô tình push nhầm file chứa mã bí mật (`.env`) lên GitHub. Tất cả đã để lại những bài học "nhớ đời" mà không một trường lớp hay sách vở nào có thể dạy được.

## 4. Bài Học Đúc Kết (Key Takeaways)

### Tư duy phát triển sản phẩm tinh gọn (Pragmatic MVP Mindset)
"Small, finished work beats big, broken ideas" (Một sản phẩm nhỏ nhưng hoàn thiện luôn chiến thắng một ý tưởng vĩ đại nhưng đổ vỡ). Đây là bài học đắt giá nhất để vượt qua áp lực thời gian: dũng cảm cắt gọt tính năng và tập trung làm thật tốt một tính năng cốt lõi (Scope it tiny).

Để có một bài thuyết trình 3 phút thành công trước ban giám khảo, việc phân chia vai trò rõ ràng (ai code, ai thiết kế, ai pitch) và chuẩn bị kỹ lưỡng kịch bản demo chính là yếu tố mang tính quyết định.

### Kiến trúc linh hoạt và tối ưu hóa chi phí vận hành (Flexible & Cost-Efficient Architecture)
Thiết kế hệ thống thông minh là thiết kế cho phép sản phẩm thay đổi, mở rộng mà không phải "đập đi xây lại" toàn bộ. Điều này đạt được nhờ việc tận dụng các adapter kết nối kênh giao tiếp và phân tách các công cụ xử lý logic độc lập.

Việc ứng dụng Framework như AgentCore không chỉ giúp cắt giảm đến 60% mã nguồn hạ tầng mà còn duy trì chi phí vận hành cực kỳ kinh tế (chỉ $0.006 cho mỗi đơn đặt hàng thông qua KFC Bot, hoặc khả năng co giãn chi phí linh hoạt dựa trên tài nguyên Serverless như dự án Signal Scout).

### Giá trị của cộng đồng và sự bổ khuyết đội nhóm (The Power of People & Diversity)
Chiến thắng lớn nhất sau một kỳ Hackathon không nằm ở giải thưởng vật chất, mà nằm ở những người đồng đội đã kề vai sát cánh chiến đấu trong đêm, cùng những mối quan hệ bền chặt được kết nối trong cộng đồng.

Sự đa dạng về thế mạnh cá nhân trong một đội nhóm (khác biệt kỹ năng luôn tốt hơn sự đồng đều kỹ năng) chính là chìa khóa để giải quyết bài toán đa chiều dưới áp lực thời gian khủng khiếp.

## 5. Ứng Dụng Vào Thực Tế (Applying to Work)
- **Áp dụng triết lý "Scope it tiny"**: Luôn ưu tiên hoàn thành dứt điểm các phiên bản sản phẩm nhỏ (MVP) để lấy phản hồi thực tế từ người dùng, tránh sa đà vào việc thiết kế những tính năng đồ sộ nhưng không bao giờ hoàn thiện.
- **Xây dựng kiến trúc hướng tác tử (Agent-based)**: Thiết kế hệ thống theo dạng module hóa, giúp dễ dàng cắm rút (plug-and-play) thêm các công cụ tự động hóa hoặc kênh phân phối mới mà không làm xáo trộn mã nguồn cốt lõi.
- **Quản trị chi phí từ trong trứng nước**: Nghiên cứu sâu phương pháp tối ưu hóa chi phí điện toán đám mây dựa trên việc đo lường tài nguyên thực tế. Tích cực sử dụng các công cụ tính toán chi phí (AWS Pricing API/Calculator) ngay từ bước phác thảo thiết kế.
- **Chuẩn bị sẵn sàng bệ phóng**: Luôn chủ động chuẩn bị sẵn các bộ khung dự án (starter templates), tài khoản Cloud và công cụ cần thiết để có thể đẩy nhanh tốc độ triển khai ngay khi có dự án mới phát sinh đột xuất.

## 6. Trải Nghiệm Khóa Học (Event Experience)

- **Trải nghiệm sống động từ diễn giả**: Những chia sẻ chân thực, mộc mạc và không hề giấu giếm về những thất bại hay sự hỗn loạn của các đội thi đã thắp sáng ngọn lửa tự tin, giúp tôi rũ bỏ nỗi sợ "không đủ năng lực" để mạnh dạn dấn thân.
- **Góc nhìn công nghệ thực tế**: Việc tận mắt chứng kiến các mô hình như Amazon Bedrock, AgentCore phối hợp cùng Computer Vision (YOLO) để tạo nên những hệ thống tự vận hành xuất sắc đã giúp tôi định hình rõ nét xu hướng công nghệ của tương lai.
- **Sử dụng công cụ thông minh**: Tôi hiểu sâu sắc hơn giá trị to lớn của AI Agent trong việc tự động hóa các tác vụ lặp đi lặp lại, nâng cao năng suất cá nhân lên gấp nhiều lần mà vẫn đảm bảo tính chính xác cao.
- **Kết nối cộng đồng**: Không khí rực lửa, hào hứng và tinh thần hỗ trợ không biên giới của cộng đồng AWS tại sự kiện đã truyền cho tôi một nguồn năng lượng mạnh mẽ để tiếp tục học hỏi và cống hiến.

## 7. Tổng Kết (Lessons Learned)
- Đừng bao giờ chờ đợi đến khi bản thân cảm thấy "hoàn toàn sẵn sàng" mới dám đăng ký tham gia các sân chơi thực chiến. Sự dũng cảm bước ra khỏi vùng an toàn mới chính là bệ phóng lớn nhất.
- Một hệ thống AI Agent xuất sắc không chỉ cần một mô hình ngôn ngữ lớn (LLM) thông minh, mà quan trọng hơn là khả năng kết nối chính xác với dữ liệu nội bộ đáng tin cậy của doanh nghiệp để đưa ra hành động thực tế.
- Hãy học cách yêu thích sự hỗn loạn, các lỗi debug lúc nửa đêm và những sản phẩm "chưa hoàn hảo". Bởi vì đó chính là nơi lưu giữ những bài học sâu sắc nhất và những kỷ niệm vô giá nhất trong cuộc đời làm kỹ thuật.

*(Hình ảnh sự kiện có thể được bổ sung tại đây)*


![Event 3](/NewsRagReport/images/event3.jpg)
