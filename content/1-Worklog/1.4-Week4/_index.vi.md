---
title: "Worklog Tuần 4"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.4. </b> "
---

### Mục tiêu tuần 4:

- Làm chủ nền tảng Serverless (Lambda, SQS, IAM Role) và tự triển khai hoàn chỉnh Luồng 1: Xử lý & Lưu trữ tài liệu.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                      | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                                                                                   |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 2   | - Tìm hiểu AWS Lambda: function, trigger, layers, environment variables, cold start, IAM execution role                                        | 13/07/2026   | 13/07/2026      | [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)                                                                  |
| 3   | - Tìm hiểu Amazon S3 (Event Notification) và Amazon SQS: Standard Queue, Dead Letter Queue (DLQ), retry policy                                 | 14/07/2026   | 14/07/2026      | [Amazon S3 Documentation](https://docs.aws.amazon.com/AmazonS3/)<br>[Amazon SQS Documentation](https://docs.aws.amazon.com/sqs/) |
| 4   | - Tìm hiểu Amazon Textract (OCR file scan/ảnh)<br>- **Thực hành:** Lambda poll message từ SQS và xử lý file                                    | 15/07/2026   | 15/07/2026      | [Amazon Textract Documentation](https://docs.aws.amazon.com/textract/)                                                           |
| 5   | - Tìm hiểu IAM Roles nâng cao: least privilege, resource-based policy giữa các service (S3 → SQS → Lambda)                                     | 16/07/2026   | 16/07/2026      | [AWS IAM Documentation](https://docs.aws.amazon.com/IAM/)                                                                        |
| 6   | - **Thực hành:** Build hoàn chỉnh Luồng 1 – S3 (upload) → S3 Event → SQS (buffer + retry) → Lambda (Document Processor) → OCR nếu là file scan | 17/07/2026   | 17/07/2026      | Dự án cá nhân                                                                                                                    |

### Kết quả đạt được tuần 4:

- Hiểu và triển khai được kiến trúc event-driven cơ bản: S3 Event → SQS → Lambda.
- Cấu hình được Dead Letter Queue và cơ chế retry khi xử lý tài liệu thất bại.
- Tự động OCR được file ảnh/scan bằng Amazon Textract.
- Phân quyền an toàn (least privilege) giữa các service bằng IAM Role.
- Có pipeline ingest tài liệu tự động, sẵn sàng để sinh vector ở tuần tiếp theo.
