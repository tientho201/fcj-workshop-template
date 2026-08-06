---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Xây dựng hệ thống hỏi đáp tài liệu thông minh với kiến trúc RAG trên AWS

#### Tổng quan

**RAG (Retrieval-Augmented Generation)** là kiến trúc kết hợp giữa khả năng tìm kiếm thông tin (retrieval) và mô hình ngôn ngữ lớn (LLM), giúp câu trả lời sinh ra bám sát vào nguồn dữ liệu thực tế thay vì chỉ dựa vào kiến thức sẵn có của mô hình.

Trong workshop này, chúng ta sẽ cùng xây dựng một hệ thống **RAG Knowledge Assistant** hoàn chỉnh trên nền tảng AWS Serverless. Hệ thống cho phép người dùng upload tài liệu (PDF/TXT/ảnh scan), tự động số hóa và lập chỉ mục ngữ nghĩa, sau đó đặt câu hỏi và nhận câu trả lời được sinh ra trực tiếp từ nội dung tài liệu đã upload — có kiểm duyệt nội dung, có bộ nhớ đệm ngữ nghĩa (semantic cache) để tối ưu chi phí, và có cơ chế tự động giám sát cùng đánh giá chất lượng câu trả lời.

Toàn bộ hệ thống được chia thành bốn luồng xử lý chính, tương ứng với bốn nhóm chức năng độc lập nhưng liên kết chặt chẽ với nhau:

- **Luồng 1 — Data Ingestion**: Thu thập tài liệu từ người dùng, xử lý OCR nếu là file scan/ảnh, chuyển nội dung thành vector và lưu trữ trong kho tìm kiếm ngữ nghĩa.
- **Luồng 2 — Hỏi đáp Realtime**: Tiếp nhận câu hỏi, tìm kiếm ngữ cảnh liên quan, sinh câu trả lời thông qua mô hình ngôn ngữ lớn, có cơ chế cache để giảm chi phí và độ trễ phản hồi.
- **Luồng 3 — Monitoring & Alert**: Giám sát toàn bộ hệ thống theo thời gian thực, phân loại cảnh báo theo mức độ nghiêm trọng và gửi thông báo tới kênh vận hành.
- **Luồng 4 — RAG Evaluation**: Tự động đo lường và đánh giá chất lượng câu trả lời theo chu kỳ hàng ngày.

#### Nội dung

1. [Tổng quan kiến trúc hệ thống](5.1-Workshop-overview/)
2. [Chuẩn bị môi trường & tài khoản AWS](5.2-Prerequisites/)
3. [Luồng 1 - Xử lý và lưu trữ tài liệu](5.3-Data-Ingestion/)
4. [Luồng 2 - Hỏi đáp Realtime với Semantic Cache](5.4-Realtime-QA/)
5. [Luồng 3 - Giám sát và cảnh báo hệ thống](5.5-Monitoring/)
6. [Luồng 4 - Đánh giá chất lượng RAG với RAGAS](5.6-RAGAS/)
7. [Frontend](5.7-Frontend/)
8. [Backend](5.8-Backend/)
9. [Xây dựng CI/CD](5.9-CICD/)
10. [Thử nghiệm hệ thống](5.10-System-Testing/)
11. [Dọn dẹp tài nguyên](5.11-Cleanup/)
