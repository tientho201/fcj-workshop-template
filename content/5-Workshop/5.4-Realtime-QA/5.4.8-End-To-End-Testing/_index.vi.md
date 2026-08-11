---
title: "Kiểm thử End-to-End"
date: 2024-01-01
weight: 8
chapter: false
pre: " <b> 5.4.8 </b> "
---

Sau khi hoàn tất hạ tầng ([5.4.1](../5.4.1-API-Gateway-Cognito),[5.4.2](../5.4.2-Cache-Guardrails-IAM/)) và toàn bộ logic xử lý ([5.4.3](../5.4.3-Cache-Lookup-Query-Rewriting/) → [5.4.7](../5.4.7-Alternative-Route/)), bước cuối cùng là kiểm thử các kịch bản thực tế của Luồng 2.

#### Kịch bản test

| #   | Kịch bản                                                        | Kỳ vọng                                                                                                   |
| --- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 1   | Câu hỏi đầu tiên của phiên mới, chưa có trong cache             | Cache miss → chạy đủ pipeline (retrieval → generation) → lưu cache + lịch sử                              |
| 2   | Hỏi lại đúng câu hỏi đó (phiên mới khác)                        | Cache hit → trả lời gần như tức thì, không thấy log gọi Bedrock                                           |
| 3   | Hỏi câu tiếp theo trong cùng phiên (có lịch sử)                 | Không dùng cache (`cacheable = False`), chạy Query Rewriting bằng Haiku                                   |
| 4   | Câu hỏi phụ thuộc ngữ cảnh (ví dụ "còn cái thứ hai thì sao?")   | Rewrite thành câu hỏi độc lập trước khi retrieval                                                         |
| 5   | Câu hỏi ngoài phạm vi tài liệu đã ingest                        | Trả "không tìm thấy tài liệu liên quan", không gọi Claude sinh câu trả lời                                |
| 6   | Câu hỏi chứa nội dung vi phạm chính sách (test Guardrail input) | Bị chặn trước khi vào retrieval                                                                           |
| 7   | Giả lập Bedrock trả về `ThrottlingException`                    | Client nhận HTTP 429 với `retryable: true`, log ERROR xuất hiện trên CloudWatch                           |
| 8   | Xác nhận OCR qua `/documents-decision` cho tài liệu đang chờ    | Nhận 202 ngay lập tức, `document_processor` được invoke bất đồng bộ, `/status` cập nhật tiến trình        |
| 9   | Bấm 👍 dưới 1 câu trả lời bình thường                           | Nút khóa + tô sáng, item mới xuất hiện trong bảng `feedback` với đúng `message_id`                        |
| 10  | Bấm 👎 dưới câu trả lời lấy từ cache hoặc bị Guardrail chặn     | Vẫn hoạt động đúng — xác nhận `message_id` được sinh cho cả 2 nhánh này (xem [5.4.7](../5.4.7-Feedback/)) |
| 11  | Bấm feedback trong lúc giả lập mất mạng                         | Nút tự mở khóa lại sau khi lỗi, có thể bấm lại được, không bị kẹt ở trạng thái "đang gửi"                 |

##### Hình 1: Kết quả test 1

![Kết quả test 1](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-1.png)

##### Hình 2: Kết quả test 2

![Kết quả test 2](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-2.png)

##### Hình 3: Kết quả test 3

![Kết quả test 3](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-3.png)

##### Hình 4: Kết quả test 4

![Kết quả test 4](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-4.png)

#### Kết quả đạt được

- API bảo mật bằng Cognito, phục vụ đủ 4 route qua 1 Lambda duy nhất, giảm số lượng function cần quản lý.
- Cache exact-match giúp giảm đáng kể chi phí/độ trễ cho các câu hỏi lặp lại đúng nguyên văn ở đầu phiên, đồng thời tránh rủi ro trả nhầm cache cho câu hỏi phụ thuộc ngữ cảnh.
- Query Rewriting bằng Claude Haiku giúp retrieval hoạt động chính xác hơn với các câu hỏi nối tiếp trong hội thoại, có fallback an toàn khi rewrite lỗi.
- Hybrid Search tự cài đặt trên DynamoDB (cosine + BM25 → RRF) thay thế hoàn toàn OpenSearch, giảm 1 thành phần hạ tầng cần vận hành.
- Guardrail 2 tầng (input/output) và xử lý Throttling riêng biệt giúp hệ thống vừa an toàn về nội dung, vừa thân thiện với client khi quá tải.
- Route `/documents-decision` kết nối liền mạch với Luồng 1, hoàn thiện trải nghiệm human-in-the-loop cho việc xác nhận OCR.
