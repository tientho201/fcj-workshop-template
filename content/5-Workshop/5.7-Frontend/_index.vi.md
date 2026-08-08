---
title: "Frontend"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

#### Giới thiệu

Frontend là giao diện web cho người dùng cuối: đăng nhập, upload tài liệu, đặt câu hỏi và xem câu trả lời. Giao tiếp backend chỉ qua **4 route API của Luồng 2** (xem [5.4.1](../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)):

| Route | Dùng để |
|---|---|
| `POST /chat` | Gửi câu hỏi, nhận câu trả lời (+ `trace`, `cached`, `sources`) |
| `POST /documents` | Upload text (`content`) hoặc nhị phân (`content_base64`) |
| `GET /status` | Poll tiến trình ingestion (`document_id`) |
| `POST /documents-decision` | Xác nhận OCR (`ocr`) hoặc hủy với PDF scan |

UI là một file trong repo ứng dụng (`ui/index.html`), gồm 2 phần chính:

- **Shell (HTML/CSS/JS)** — bảng điều khiển 2 pane. Không React, không bundler, không `package.json`.
  - **Pane trái:** (1) Kết nối — API URL, Cognito App Client ID, region, email/mật khẩu; (2) Upload — chọn file hoặc dán text, hộp Có/Không OCR khi cần; (3) Chat — hội thoại, ô câu hỏi, **Phiên mới** (`session_id`).
  - **Pane phải:** tab luồng nạp / hỏi đáp, animation từng bước theo thời gian thật backend, kèm nhật ký.
- **Tích hợp Cognito và API Gateway** — trình duyệt gọi Cognito `InitiateAuth` (`USER_PASSWORD_AUTH`) lấy ID token, rồi gọi bốn route với `Authorization: <IdToken>`. Thời gian pane phải lấy từ `trace` của `/chat` hoặc tiến trình `/status` — không giả lập.
  {{% notice note %}}
  📌 **Console demo, chỉ chạy local.** Stack **không** host file trên Amplify hay S3 static website — mở `ui/index.html` sau Terraform apply (xem [5.7.3](5.7.3-Deployment-Hosting/)).

  📌 UI ghi bước Redis là “Semantic cache”, nhưng triển khai thực tế là cache **exact-match** (hash câu hỏi đã chuẩn hóa) — cùng lưu ý như Luồng 2. ElastiCache Serverless ở đây không có module RediSearch/vector.

  📌 Terraform mặc định `api_require_api_key = true`, nhưng `ui/index.html` **không** gửi `x-api-key`. Để UI local chạy được: đặt `api_require_api_key = false` trong `terraform.tfvars`, hoặc bổ sung UI gửi `terraform output -raw api_key_value`.
  {{% /notice %}}

#### Giao diện UI

![Giao diện UI](/images/5-Workshop/5.7-Web/image.png)

#### Nội dung chi tiết

1. [Kiến trúc Frontend và Authentication](5.7.1-Frontend-Architecture-Authentication/)
2. [Giao diện Chat và Upload tài liệu](5.7.2-Chat-Upload-UI/)
3. [Triển khai và Hosting](5.7.3-Deployment-Hosting/)
4. [Kiểm thử end-to-end](5.7.4-Test-end-to-end/)
