---
title: "Frontend"
date: 2026-08-07
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

#### Giao diện điều khiển RAG

Khác với SPA có build step thường gặp, giao diện dự án là **một file HTML/JS thuần tự chứa** (`ui/index.html`) — **không React, không bundler, không `package.json`**. Lựa chọn này khớp quy mô: bảng điều khiển nội bộ để demo/kiểm thử hệ thống RAG, không phải sản phẩm public cần SEO hay code-splitting.

Trình duyệt chỉ gọi backend qua **4 route API của Luồng 2** (xem [5.4.1](../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)):

| Route | Dùng để |
|---|---|
| `POST /chat` | Gửi câu hỏi, nhận câu trả lời (+ `trace`, `cached`, `sources`) |
| `POST /documents` | Upload text (`content`) hoặc nhị phân (`content_base64`) |
| `GET /status` | Poll tiến trình ingestion (`document_id`) |
| `POST /documents-decision` | Xác nhận OCR (`ocr`) hoặc hủy với PDF scan |

Giao diện chia **2 cột**: **trái** là khu thao tác (đăng nhập, tải tài liệu, hỏi đáp), **phải** là khu quan sát pipeline — sơ đồ các bước chạy animation **theo đúng số liệu thời gian thật** backend trả về (`trace` / trạng thái ingestion), không phải hiệu ứng dàn dựng.

{{% notice note %}}
📌 **Console demo, chỉ chạy local.** Stack **không** host file trên Amplify hay S3 static website — mở `ui/index.html` sau Terraform apply (xem [5.7.3](5.7.3-Deployment-Hosting/)).

📌 UI ghi bước Redis là “Semantic cache”, nhưng triển khai thực tế là cache **exact-match** (hash câu hỏi đã chuẩn hóa) — cùng lưu ý như Luồng 2. ElastiCache Serverless ở đây không có module RediSearch/vector.

📌 Terraform mặc định `api_require_api_key = true`, nhưng `ui/index.html` **không** gửi `x-api-key`. Để UI local chạy được: đặt `api_require_api_key = false` trong `terraform.tfvars`, hoặc bổ sung UI gửi `terraform output -raw api_key_value`.
{{% /notice %}}

#### Sơ đồ giao diện

![Giao diện 2 cột: khu thao tác và khu quan sát pipeline](/images/5-Workshop/5.7-Web/image.png)
*Ảnh chụp console 2 cột thật (trái: kết nối / upload / chat; phải: pipeline + nhật ký).*

#### Nội dung chi tiết

1. [Kiến trúc Frontend và Authentication](5.7.1-Frontend-Architecture-Authentication/)
2. [Giao diện Chat và Upload tài liệu](5.7.2-Chat-Upload-UI/)
3. [Triển khai và Hosting](5.7.3-Deployment-Hosting/)
4. [Kiểm thử end-to-end](5.7.4-Test-end-to-end/)
