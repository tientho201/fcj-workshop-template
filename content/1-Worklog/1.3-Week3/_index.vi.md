---
title: "Worklog Tuần 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu tuần 3:

- Họp nhóm để thảo luận, đề xuất và chốt đề tài cho dự án cá nhân.

- Lên ý tưởng và đảm bảo đề tài giải quyết một bài toán thiết thực trên AWS.

- Nghiên cứu và lựa chọn kiến trúc hạ tầng phù hợp (ưu tiên Serverless) để xây dựng hệ thống.

- Hoàn thiện Proposal với định hướng công nghệ và lộ trình phát triển rõ ràng.

- Trực quan hóa giải pháp bằng các sơ đồ kiến trúc (Architecture Diagram) chi tiết.

- Chuẩn bị môi trường mã nguồn, khắc phục các sự cố liên quan đến template dự án (lỗi Git submodule).

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                                                                                                                                               |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2   | **Họp nhóm và lên ý tưởng dự án:**<br>- Họp nhóm để cùng thảo luận, đề xuất và định hướng đề tài.<br>- Phân tích bài toán tra cứu và hỏi đáp tri thức nội bộ (thông tin phân tán trong nhiều tài liệu PDF/scan, khó tìm kiếm nhanh, tốn thời gian tra cứu thủ công).<br>- Tham khảo Amazon Bedrock Knowledge Bases và kiến trúc RAG (Retrieval-Augmented Generation) để định vị ý tưởng.<br>- Thống nhất và chốt đề tài dự án nhóm RAG Knowledge Assistant.                                                                                                                           | 06/07/2026   | 06/07/2026      | [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)<br>[What is RAG?](https://aws.amazon.com/what-is/retrieval-augmented-generation/) |
| 3   | **Nghiên cứu và thiết kế hệ thống:**<br>- Trao đổi với nhóm về kế hoạch công việc trong ngày trước khi bắt đầu.<br>- Tìm hiểu ưu điểm của kiến trúc Serverless cho hệ thống GenAI (tự động scale theo tải, tối ưu chi phí khi không có traffic, không cần quản lý server).<br>- Thiết kế 4 luồng chính: Data Ingestion, Hỏi đáp Realtime (Semantic Cache), Monitoring/Alert, RAG Evaluation (RAGAS).<br>- Chốt công nghệ triển khai: Serverless (Lambda, SQS, Bedrock, OpenSearch Serverless, DynamoDB) kết hợp IaC (Terraform).<br>- Cuối ngày tổng hợp và chia sẻ kết quả với nhóm. | 07/07/2026   | 07/07/2026      | [AWS Serverless Architecture](https://aws.amazon.com/serverless/)<br> AWS Well-Architected – GenAI Lens                                                                                      |
| 4   | **Viết Proposal dự án:**<br>- Trao đổi với nhóm về kế hoạch công việc trong ngày trước khi bắt đầu.<br>- Viết phần giới thiệu, phạm vi và mục tiêu dự án (chatbot hỏi đáp tài liệu nội bộ có kiểm duyệt nội dung và đo lường chất lượng câu trả lời).<br>- Lập luận lý do chọn đề tài dựa trên nhu cầu thực tế doanh nghiệp và mục đích học tập GenAI/Serverless.<br>- Định nghĩa các use case cụ thể: upload tài liệu, hỏi đáp real-time, thu thập feedback, đánh giá chất lượng RAG tự động.<br>- Cuối ngày tổng hợp và chia sẻ tiến độ với nhóm.                                   | 08/07/2026   | 08/07/2026      | Tổng hợp kiến thức từ các tài liệu nghiên cứu thiết kế hệ thống RAG và Bedrock.                                                                                                              |
| 5   | **Vẽ Diagram:**<br>- Trao đổi với nhóm về kế hoạch công việc trong ngày trước khi bắt đầu.<br>- Vẽ sơ đồ kiến trúc tổng quan (High-level Architecture) gồm 4 luồng: Ingestion, Realtime QA, Monitoring, Evaluation.<br>- Vẽ sơ đồ luồng dữ liệu (Data Flow) mô tả cách các thành phần Serverless (S3, SQS, Lambda, Bedrock, OpenSearch, DynamoDB, ElastiCache) tương tác để xử lý tài liệu và trả lời câu hỏi.<br>- Cuối ngày tổng hợp và chia sẻ kết quả với nhóm.                                                                                                                   | 09/07/2026   | 09/07/2026      |                                                                                                                                                                                              |
| 6   | **Khởi tạo môi trường:**<br>- Trao đổi với nhóm về kế hoạch công việc trong ngày trước khi bắt đầu.<br>- Kiểm tra và rà soát cấu trúc thư mục dự án ban đầu (repo Git, cấu trúc Terraform/IaC).<br>- Gửi yêu cầu cấp quyền truy cập model trên Amazon Bedrock (model access request cho Claude 3, Titan Embeddings).<br>- Xử lý các lỗi cấu hình ban đầu (Git submodule, quyền IAM cho tài khoản nhóm, kết nối GitHub).<br>- Cấu hình lại chuẩn xác để sẵn sàng cho quá trình phát triển.<br>- Cuối ngày tổng hợp và chia sẻ tiến độ với nhóm.                                        | 10/07/2026   | 10/07/2026      |                                                                                                                                                                                              |

### Kết quả đạt được tuần 3:

- **Họp nhóm và chốt ý tưởng dự án:** Tuần làm việc bắt đầu bằng buổi họp nhóm để cùng nhau thảo luận, đề xuất và định hướng đề tài. Sau khi trao đổi, nhóm đã hoàn thành việc lên ý tưởng và chốt đề tài dự án RAG Knowledge Assistant, một hệ thống chatbot hỏi đáp tài liệu nội bộ dựa trên kiến trúc RAG. Quá trình lên ý tưởng xuất phát từ một bài toán thực tế: thông tin trong doanh nghiệp thường phân tán ở nhiều tài liệu PDF/scan khác nhau, gây tốn thời gian tra cứu thủ công và khó tận dụng tri thức có sẵn.

- **Lý do lựa chọn đề tài:** Trong quá trình nghiên cứu, nhóm đã tham khảo Amazon Bedrock Knowledge Bases, dịch vụ managed cho phép xây dựng RAG có sẵn của AWS. Việc này giúp khẳng định RAG là một nhu cầu thực tế được chính AWS công nhận trong lĩnh vực GenAI. Trên cơ sở đó, nhóm lựa chọn hướng đi cốt lõi là thay vì dùng dịch vụ managed sẵn có, dự án sẽ tự xây dựng toàn bộ pipeline ingestion, retrieval, sinh câu trả lời và đánh giá chất lượng bằng kiến trúc Serverless kết hợp Terraform (IaC). Hướng tiếp cận này nhằm ba mục đích: học tập và làm chủ hoàn toàn kiến trúc GenAI Serverless trên AWS, rèn luyện kỹ năng Infrastructure as Code, và chủ động tùy biến logic retrieval, caching, kiểm duyệt nội dung theo nhu cầu riêng.

- **Hoàn thiện bộ tài liệu đề xuất:** Nhóm đã viết xong Proposal dự án rõ ràng, chi tiết hóa mục tiêu, phạm vi và định hướng công nghệ. Đồng thời hoàn thành sơ đồ kiến trúc tổng quan và luồng dữ liệu cho cả 4 luồng chính (Ingestion, Realtime QA, Monitoring, RAGAS Evaluation), giúp trực quan hóa cách các dịch vụ Serverless tương tác với nhau.

- **Xử lý sự cố môi trường phát triển:** Khắc phục thành công các lỗi cấu hình ban đầu trong repo/template, gửi yêu cầu cấp quyền truy cập model trên Bedrock, thiết lập môi trường phát triển đồng bộ cho cả nhóm và sẵn sàng bước vào giai đoạn lập trình, cấu hình hạ tầng ở các tuần tiếp theo.

- **Phối hợp cùng nhóm:** Duy trì thói quen làm việc nhóm hiệu quả trong suốt tuần. Trước khi bắt đầu công việc mỗi ngày, các thành viên trao đổi kế hoạch với nhau, và cuối mỗi ngày tổng hợp lại kết quả đã làm để cả nhóm cùng nắm tiến độ.
