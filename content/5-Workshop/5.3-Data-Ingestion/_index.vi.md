---
title: "Xử lý và lưu trữ tài liệu"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

#### Giới thiệu

Luồng 1 đảm nhiệm vai trò đầu vào của toàn bộ hệ thống — chuyển đổi tài liệu thô do người dùng upload (PDF, ảnh scan, file text) thành dữ liệu có cấu trúc, có thể truy vấn theo ngữ nghĩa, làm nền tảng cho Luồng 2 (Hỏi đáp Realtime) khai thác sau này.

Luồng vận hành theo mô hình event-driven, gồm 2 phần chính:

- **Hạ tầng (Terraform)** — S3 lưu tài liệu gốc, SQS làm buffer trung gian với 2 tầng Dead Letter Queue, và 3 bảng DynamoDB lưu trạng thái xử lý cùng dữ liệu vector/BM25 (thay thế hoàn toàn OpenSearch Serverless trong sơ đồ kiến trúc ban đầu).
- **Logic xử lý (Lambda `handler.py`)** — trích xuất văn bản theo từng loại file, chia nhỏ tài liệu theo chiến lược Parent-Child chunking, sinh vector embedding qua Amazon Bedrock, và có cơ chế xác nhận OCR thủ công cho tài liệu dạng scan.

#### Sơ đồ luồng dữ liệu

![Sơ đồ chi tiết Luồng 1 - Xử lý và lưu trữ tài liệu](/images/5-Workshop/5.3-Data-Ingestion/image.png)

#### Nội dung chi tiết

Các phần triển khai chi tiết của luồng này được trình bày ở các trang con dưới đây:

1. [Hạ tầng: S3 và SQS](5.3.1-Infrastructure-S3-SQS/)
2. [Hạ tầng: DynamoDB và IAM Permissions](5.3.2-Infrastructure-DynamoDB-IAM/)
3. [Trích xuất văn bản theo loại file](5.3.3-Text-Extraction/)
4. [Parent-Child Chunking và Embedding](5.3.4-Chunking-Embedding/)
5. [Cơ chế Resume OCR và xử lý lỗi](5.3.5-Resume-OCR-Error-Handling/)
6. [Kiểm thử end-to-end](5.3.6-End-To-End-Testing/)
