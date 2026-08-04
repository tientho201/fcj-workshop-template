---
title: "Giới thiệu"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

#### Nguyên lý thiết kế

Hệ thống **RAG Knowledge Assistant** được thiết kế theo bốn nguyên tắc chính:

- **Serverless-first** — không quản lý máy chủ, tự động scale theo tải, chỉ trả phí theo mức sử dụng thực tế (S3, Lambda, SQS, DynamoDB, OpenSearch Serverless, ElastiCache Serverless đều là dịch vụ serverless/managed hoàn toàn).
- **Event-driven** — các thành phần giao tiếp với nhau thông qua sự kiện (S3 Event, SQS message) thay vì gọi trực tiếp đồng bộ, giúp hệ thống chịu lỗi tốt hơn và dễ mở rộng độc lập từng phần.
- **Tách biệt mối quan tâm (Separation of Concerns)** — bốn luồng xử lý hoạt động độc lập, mỗi luồng có thể phát triển, kiểm thử và triển khai riêng mà không ảnh hưởng luồng khác.
- **Quan sát được (Observable) ngay từ đầu** — mọi thành phần đều đẩy log/metric về CloudWatch, không đợi đến khi có sự cố mới bổ sung giám sát.
  Toàn bộ hạ tầng được quản lý bằng **Terraform (Infrastructure as Code)**, đảm bảo khả năng tái tạo môi trường (dev/staging/prod) nhất quán và có thể review thay đổi hạ tầng qua Pull Request.

#### Kiến trúc tổng quan

<div align="center">

![Kiến trúc tổng quan hệ thống RAG Knowledge Assistant](/images/5-Workshop/5.1-Workshop-overview/aws-new.drawio.png)

<p style="font-size: 0.85em; color: #666; font-style: italic; margin-top: 8px;">
 Kiến trúc tổng quan hệ thống RAG Knowledge Assistant
</p>

</div>

Kiến trúc gồm bốn luồng xử lý chính, hoạt động độc lập nhưng liên kết chặt chẽ với nhau qua dữ liệu chia sẻ (vector index, chat history). Chi tiết từng luồng sẽ được trình bày riêng ở các chương tiếp theo — phần này chỉ tóm tắt vai trò và thành phần chính để có cái nhìn tổng thể trước khi đi vào triển khai.

| Luồng                     | Vai trò                                                           | Thành phần chính                                                                                                           |
| ------------------------- | ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **1. Data Ingestion**     | Thu thập, số hóa (OCR) và lập chỉ mục ngữ nghĩa cho tài liệu      | S3, SQS (+ DLQ), Lambda, Textract, Bedrock (Embedding), OpenSearch Serverless                                              |
| **2. Hỏi đáp Realtime**   | Nhận câu hỏi, truy xuất ngữ cảnh, sinh câu trả lời, cache kết quả | API Gateway, Cognito, Lambda, ElastiCache Serverless, OpenSearch Serverless, Bedrock (Claude/Titan + Guardrails), DynamoDB |
| **3. Monitoring & Alert** | Giám sát hệ thống, cảnh báo theo mức độ nghiêm trọng              | CloudWatch (Logs, Metrics, Alarms, Dashboard), SNS, AWS Chatbot, Slack                                                     |
| **4. RAG Evaluation**     | Tự động chấm điểm chất lượng câu trả lời hàng ngày                | EventBridge Scheduler, Lambda (RAGAS Runner), S3 (Evaluation Results)                                                      |

#### Bảng tổng hợp dịch vụ AWS sử dụng

| Nhóm                | Dịch vụ                                                       |
| ------------------- | ------------------------------------------------------------- |
| Compute             | AWS Lambda                                                    |
| Storage             | Amazon S3                                                     |
| Messaging           | Amazon SQS, Amazon SNS                                        |
| Database            | Amazon DynamoDB, Amazon ElastiCache Serverless                |
| Search & AI         | Amazon OpenSearch Serverless, Amazon Bedrock, Amazon Textract |
| API & Bảo mật       | Amazon API Gateway, Amazon Cognito, IAM                       |
| Giám sát & Vận hành | Amazon CloudWatch, AWS Chatbot                                |
| Điều phối           | Amazon EventBridge Scheduler                                  |

Ở chương tiếp theo, chúng ta sẽ chuẩn bị môi trường và tài khoản AWS cần thiết trước khi bắt tay triển khai chi tiết từng luồng.
