---
title: "Frontend"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

#### Giới thiệu

Frontend là giao diện web để người dùng cuối tương tác với hệ thống: đăng nhập, upload tài liệu, đặt câu hỏi và xem câu trả lời. Frontend giao tiếp với backend hoàn toàn qua **4 route API đã xây dựng ở Luồng 2** (xem lại [trang 5.4.1](../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)):

| Route                      | Dùng để                                 |
| -------------------------- | --------------------------------------- |
| `POST /chat`               | Gửi câu hỏi, nhận câu trả lời           |
| `POST /documents`          | Upload tài liệu mới                     |
| `GET /status`              | Poll tiến trình xử lý tài liệu          |
| `POST /documents-decision` | Xác nhận/hủy OCR cho tài liệu dạng scan |

#### Giao diện điều khiển RAG

Giao diện của dự án là **một file HTML/JS thuần tự chứa** (`ui/index.html`) — **không React, không bundler, không `package.json`**. Lựa chọn này phù hợp với quy mô dự án: một bảng điều khiển nội bộ để demo/kiểm thử hệ thống RAG, không phải sản phẩm public cần SEO hay code-splitting.

#### Giao diện UI

![Giao diện UI](/images/5-Workshop/5.7-Web/image.png)

#### Nội dung chi tiết

1. [Kiến trúc Frontend và Authentication](5.7.1-Frontend-Architecture-Authentication/)
2. [Giao diện Chat và Upload tài liệu](5.7.2-Chat-Upload-UI/)
3. [Triển khai và Hosting](5.7.3-Deployment-Hosting/)
4. [Kiểm thử end-to-end](5.7.4-Test-end-to-end/)
