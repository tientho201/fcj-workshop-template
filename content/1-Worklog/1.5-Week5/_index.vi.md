---
title: "Worklog Tuần 5"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu tuần 5:

- Mở cổng giao tiếp bảo mật cho người dùng cuối, kết nối được retrieval từ OpenSearch, và triển khai Semantic Cache để giảm chi phí gọi Bedrock.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                                                                                                                                                                             | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                                                                                                                                           |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2   | - Trao đổi kế hoạch đầu ngày với nhóm.<br>- Dựng Amazon API Gateway (REST API) với resource `/chat`, method `POST`, tích hợp kiểu Lambda Proxy Integration để toàn quyền kiểm soát response format trong code.<br>- Cấu hình CORS — bổ sung `OPTIONS` method thủ công cho preflight request (khắc phục lỗi thiếu header `Access-Control-Allow-Origin` ban đầu).                       | 20/07/2026   | 20/07/2026      | [Amazon API Gateway Documentation](https://docs.aws.amazon.com/apigateway)                                                                                                               |
| 3   | - Sync nhanh với nhóm trước khi bắt đầu.<br>- Tìm hiểu Amazon Cognito: tạo User Pool, App Client (không dùng client secret vì gọi từ frontend), gắn Cognito Authorizer vào API Gateway.<br>- Test đăng nhập lấy ID Token, gọi API kèm header `Authorization: Bearer <token>` (xác thực bằng ID Token/Access Token).                                                                   | 21/07/2026   | 21/07/2026      | [Amazon Cognito Documentation](https://docs.aws.amazon.com/cognito)                                                                                                                      |
| 4   | - Thống nhất với nhóm mục tiêu ngày hôm nay.<br>- Nghiên cứu Amazon ElastiCache Serverless & Semantic Cache (tính embedding câu hỏi mới, so sánh cosine similarity với ngưỡng threshold `0.92`).<br>- Thiết kế luồng: Câu hỏi → Embedding → Tra Redis (ElastiCache) → Nếu miss mới đi tiếp retrieval + Bedrock.                                                                       | 22/07/2026   | 22/07/2026      | [Amazon ElastiCache Documentation](https://docs.aws.amazon.com/elasticache)                                                                                                              |
| 5   | - Báo cáo nhanh tiến độ với nhóm đầu ngày.<br>- Thiết kế schema DynamoDB: bảng `ChatHistory` (Partition Key `session_id`, Sort Key `timestamp`), bảng `FeedbackStore` (gắn `message_id` với thumbs up/down).<br>- Cấu hình Bedrock Guardrails: policy chặn chủ đề nhạy cảm và mask thông tin cá nhân (PII như SĐT, email).                                                            | 23/07/2026   | 23/07/2026      | [Amazon DynamoDB Documentation](https://docs.aws.amazon.com/dynamodb)<br>[Amazon Bedrock Guardrails Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html) |
| 6   | - Ráp nối toàn bộ trong ngày cuối tuần.<br>- Hoàn thiện Lambda Chat Engine theo luồng: API Gateway → Check Semantic Cache → Miss thì query OpenSearch → Gọi Bedrock qua Guardrails → Lưu cache → Ghi DynamoDB → Trả response.<br>- Test Postman 10 câu hỏi mẫu (3 câu diễn đạt lại đều cache hit, latency giảm từ ~4s xuống < 200ms).<br>- Cuối ngày tổng kết và demo nhanh cho nhóm. | 24/07/2026   | 24/07/2026      | Dự án cá nhân                                                                                                                                                                            |

### Kết quả đạt được tuần 5:

- **Hoàn thiện cổng giao tiếp bảo mật:** API Gateway kết hợp Cognito Authorizer đảm bảo chỉ người dùng đã đăng nhập mới truy cập được chatbot.
- **Semantic Cache hoạt động đúng thiết kế:** Giảm thời gian phản hồi hơn 20 lần với các câu hỏi trùng ý (từ ~4s xuống dưới 200ms), đồng nghĩa giảm tương ứng số lần gọi Bedrock và chi phí đi kèm.
- **Lưu trữ dữ liệu lịch sử & feedback:** Dữ liệu lịch sử hội thoại (`ChatHistory`) và feedback (`FeedbackStore`) đã có nơi lưu trữ có cấu trúc trong DynamoDB, sẵn sàng phục vụ cho việc đánh giá chất lượng RAG ở tuần cuối.
- **Bedrock Guardrails hoạt động hiệu quả:** Guardrails được bật đúng vị trí trong luồng xử lý (trước khi trả kết quả về người dùng), đảm bảo lọc chủ đề nhạy cảm và mask thông tin cá nhân (PII).
- **Luồng 2 (Hỏi đáp Realtime) chạy ổn định end-to-end:** Phần trải nghiệm người dùng trực tiếp nhất của toàn dự án đã được tích hợp và kiểm thử kỹ lưỡng qua Postman với nhiều kịch bản khác nhau.
