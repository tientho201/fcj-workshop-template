---
title: "Hỏi đáp Realtime với Semantic Cache"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

#### Giới thiệu

Luồng 2 tiếp nhận câu hỏi từ người dùng, truy xuất ngữ cảnh liên quan từ dữ liệu đã lập chỉ mục ở Luồng 1, sinh câu trả lời qua Amazon Bedrock, và trả về cho người dùng — có kiểm duyệt nội dung, có cache để giảm chi phí, và lưu lịch sử phục vụ đánh giá chất lượng ở Luồng 4.

Luồng vận hành với **1 Lambda `chat_engine` duy nhất phục vụ 4 route API**, gồm 2 phần chính:

- **Hạ tầng (Terraform)** — API Gateway với 4 route dùng chung 1 Lambda, Cognito xác thực người dùng, ElastiCache Serverless làm cache, Bedrock Guardrails kiểm duyệt nội dung, và IAM Role least-privilege.
- **Logic xử lý (Lambda `handler.py`)** — rẽ nhánh theo route, với route chính `_handle_chat` chạy tuần tự: cache lookup → query rewriting → guardrail đầu vào → hybrid search (DynamoDB) → sinh câu trả lời → guardrail đầu ra → ghi cache/lịch sử.
  {{% notice note %}}
  📌 Điểm khác biệt quan trọng so với tên gọi ban đầu: **ElastiCache ở đây là cache exact-match** (hash câu hỏi → câu trả lời, có TTL), **không phải cache ngữ nghĩa thật** — vì ElastiCache Serverless Redis không hỗ trợ module RediSearch/vector search. Biến `cache_similarity_threshold` hiện chỉ để dành cho hướng nâng cấp sau này (MemoryDB hoặc self-managed Redis+RediSearch).
  {{% /notice %}}

#### Sơ đồ luồng dữ liệu

![Sơ đồ chi tiết Luồng 2 - Hỏi đáp Realtime](/images/5-Workshop/5.4-Realtime-QA/image.png)

#### Nội dung chi tiết

1. [Hạ tầng: API Gateway và Cognito](5.4.1-API-Gateway-Cognito/)
2. [Hạ tầng: Semantic Cache, Guardrails và IAM](5.4.2-Cache-Guardrails-IAM/)
3. [Cache lookup và Query Rewriting](5.4.3-Cache-Lookup-Query-Rewriting/)
4. [Hybrid Search và Retrieval](5.4.4-Hybrid-Search-Retrieval/)
5. [Sinh câu trả lời và lưu lịch sử](5.4.5-Answer-Generation-History-Storage/)
6. [Xử lý lỗi và kết nối OCR Decision](5.4.6-Error-Handling-OCR-Decision/)
7. [Route phụ](5.4.7-Alternative-Route/-Error-Handling-OCR-Decision/)
8. [Kiểm thử end-to-end](5.4.8-End-To-End-Testing/)
