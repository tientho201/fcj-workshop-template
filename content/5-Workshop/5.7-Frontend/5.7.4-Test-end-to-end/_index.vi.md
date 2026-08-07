---
title: "Kiểm thử end-to-end"
date: 2026-08-07
weight: 4
chapter: false
pre: " <b> 5.7.4. </b> "
---

Sau [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)–[5.7.3](../5.7.3-Deployment-Hosting/), kiểm tra UI với API đang chạy. Mở `ui/index.html`, điền kết nối từ `terraform output`, đăng nhập user Cognito.

#### Kịch bản test

| # | Thao tác | Kỳ vọng |
|---|---|---|
| 1 | Đăng nhập user Cognito hợp lệ | Đèn auth xanh; Upload và Gửi bật; nhận ID token |
| 2 | Sai mật khẩu | Lỗi trong log; nút vẫn tắt |
| 3 | Upload `.txt` / `.md` có nội dung quen, chờ ingest xong | Pane phải hiện bước nạp; summary có số parent/child |
| 4 | Hỏi về nội dung tài liệu đó | Trả lời bám nội dung; có tag nguồn nếu có; luồng hỏi đáp animate |
| 5 | Bấm **Phiên mới**, hỏi lại **y hệt** | Tag `cache hit`; chủ yếu 1 bước sáng; phản hồi dưới 1 giây |
| 6 | Hỏi thứ không có trong corpus | Model từ chối / không bịa tự do |
| 7 | Câu follow-up mơ hồ trong **cùng phiên** (“còn cái kia thì sao?”) | Bước query rewriting sáng; log hiện câu đã viết lại |
| 8 | Upload PDF scan / không lớp text | `awaiting_ocr_confirmation` → hộp Có/Không → OCR hoặc huỷ |
| 9 | Gửi nhiều câu liên tiếp nhanh | Có thể HTTP 429 `retryable` — đợi rồi thử lại (đúng với quota Bedrock thấp) |

{{% notice tip %}}
Nếu ingest quá ~90s, xem CloudWatch Logs `document-processor` và độ sâu SQS/DLQ (alarm Luồng 3). Timeout UI cố ý để pipeline kẹt lộ ra thay vì quay mãi.
{{% /notice %}}

#### Kết quả đạt được

- UI một file gọi đủ bốn route Luồng 2 với JWT Cognito thật.
- Upload phân biệt text và nhị phân (base64) cho Textract.
- Animation pipeline phản ánh `trace` / status server, kể cả bước skipped.
- Cache hit và rewrite multi-turn nhìn thấy được mà không cần mở AWS Console.

#### Checklist hoàn thành Frontend

- [ ] Đã dán `terraform output` vào **1 · Kết nối**
- [ ] Đăng nhập Cognito thành công; mật khẩu không nằm trong `localStorage`
- [ ] Upload text hoàn tất và trả lời chat đúng nội dung
- [ ] Đã xác nhận đường cache hit sau **Phiên mới**
- [ ] Hiểu luồng xác nhận OCR (kể cả khi không demo mọi lần)
- [ ] Rõ frontend chỉ chạy local (không Amplify/S3 website trong stack này)
