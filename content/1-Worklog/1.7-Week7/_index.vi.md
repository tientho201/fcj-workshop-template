---
title: "Worklog Tuần 7"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.7. </b> "
---

### Mục tiêu tuần 7:

- Rà soát lại toàn bộ 4 luồng dưới góc nhìn tổng thể, xử lý các điểm còn thô ráp phát hiện ở tuần trước (đặc biệt là chất lượng retrieval), viết tài liệu vận hành, và demo chính thức trước nhóm/mentor.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                                                                                                                                                                                                                                           | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------------------------------------------------------------------- |
| 2   | **Cải thiện chất lượng Retrieval:**<br>- Đầu tuần họp với nhóm rà lại vấn đề Faithfulness thấp phát hiện tuần trước.<br>- Điều chỉnh lại tham số Hybrid Search trong OpenSearch (tăng trọng số cho vector search so với BM25 từ tỷ lệ 50/50 lên 70/30).<br>- Giảm kích thước chunk con từ 300 xuống 200 token để tăng độ chính xác khi tìm kiếm.<br>- Chạy lại 5 câu hỏi từng bị điểm thấp — 4/5 câu đã cải thiện rõ rệt.                           | 03/08/2025   | 03/08/2025      | Dự án cá nhân                                                                                         |
| 3   | **Kiểm thử tải (load test) sơ bộ & Rà soát IAM:**<br>- Dùng script gửi 50 request đồng thời tới API Gateway để xem hệ thống phản ứng ra sao.<br>- Phát hiện ElastiCache Serverless xử lý tốt nhưng Lambda Chat Engine bị giới hạn concurrency mặc định (1000), không phải vấn đề thực tế ở quy mô hiện tại nhưng ghi chú lại để lưu ý khi scale.<br>- Rà soát lại toàn bộ IAM Policy theo nguyên tắc least privilege lần cuối trước khi hoàn thiện. | 04/08/2025   | 04/08/2025      | [AWS Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html) |
| 4   | **Hoàn thiện Terraform & IaC:**<br>- Dọn dẹp lại toàn bộ mã Terraform, tách module rõ ràng theo từng luồng (ingestion, chat-api, monitoring, evaluation), viết README hướng dẫn deploy từ đầu.<br>- Đây là phần khá tốn thời gian vì trong lúc làm gấp ở các tuần trước, một số resource được tạo thủ công qua Console chưa được đưa vào code — phải rà lại và import ngược vào Terraform state.                                                    | 05/08/2025   | 05/08/2025      | [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)                             |
| 5   | **Viết tài liệu vận hành & runbook:**<br>- Soạn tài liệu mô tả kiến trúc tổng thể, hướng dẫn xử lý khi có alert (ví dụ: DLQ Depth > 0 thì kiểm tra message lỗi ở đâu, cách requeue thủ công), và checklist bảo trì định kỳ.<br>- Chuẩn bị slide và kịch bản demo cho buổi trình bày cuối kỳ.                                                                                                                                                        | 06/08/2025   | 06/08/2025      | Dự án cá nhân                                                                                         |
| 6   | **Demo chính thức & tổng kết:**<br>- Trình bày toàn bộ dự án trước nhóm/mentor: chạy trực tiếp từ upload tài liệu → hỏi đáp → xem alert giả lập → xem báo cáo RAGAS.<br>- Ghi nhận góp ý (chủ yếu xoay quanh việc mở rộng bộ câu hỏi đánh giá và cân nhắc thêm rate-limiting cho API).<br>- Tổng kết lại toàn bộ hành trình 5 tuần và note các hướng phát triển tiếp theo.                                                                          | 07/08/2025   | 07/08/2025      | Dự án cá nhân                                                                                         |

### Kết quả đạt được tuần 7 — và tổng kết dự án:

- **Tinh chỉnh hệ thống dựa trên chỉ số thực tế:** Việc điều chỉnh tỷ lệ Hybrid Search (tăng trọng số vector search so với BM25 từ 50/50 lên 70/30) và giảm kích thước chunk con (300 → 200 tokens) là minh chứng rõ cho giá trị của vòng lặp đánh giá RAGAS đã xây ở tuần 6. Nhờ có con số định lượng cụ thể để đối chiếu, 4/5 câu hỏi từng bị điểm thấp đã cải thiện rõ rệt.
- **Kiểm thử tải & Tối ưu bảo mật:** Kiểm thử tải sơ bộ với 50 request đồng thời khẳng định ElastiCache Serverless xử lý tốt, đồng thời giúp phát hiện giới hạn concurrency mặc định của Lambda Chat Engine để lưu ý khi scale. Rà soát lại toàn bộ IAM Policy theo nguyên tắc least privilege.
- **Hoàn thiện Infrastructure as Code (IaC):** Dọn dẹp mã Terraform, tách module rõ ràng theo 4 luồng hệ thống (ingestion, chat-api, monitoring, evaluation), import ngược các tài nguyên tạo thủ công qua Console vào Terraform state và viết README hướng dẫn deploy từ đầu.
- **Tài liệu vận hành & Demo thành công:** Soạn thảo xong tài liệu vận hành (runbook mô tả kiến trúc, cách xử lý alert như DLQ Depth > 0, checklist bảo trì). Trình bày demo chính thức trôi chảy từ upload tài liệu, hỏi đáp, xem alert giả lập đến xem báo cáo RAGAS, ghi nhận phản hồi tích cực từ mentor/nhóm.

---

### Tổng kết hành trình 5 tuần dự án RAG Knowledge Assistant:

Tuần cuối này không phát sinh nhiều kiến thức mới, nhưng lại là tuần "vất vả" theo một cách khác — quay lại rà soát những gì đã làm vội ở các tuần trước để đưa hệ thống về trạng thái ổn định, có thể vận hành lâu dài chứ không chỉ chạy được cho demo. Việc điều chỉnh tỷ lệ Hybrid Search và giảm kích thước chunk là một minh chứng rõ cho giá trị của vòng lặp đánh giá RAGAS đã xây ở tuần 6 — nếu không có con số cụ thể để đối chiếu, rất khó biết nên tinh chỉnh theo hướng nào.

Nhìn lại toàn bộ 5 tuần triển khai (từ kickoff ở Tuần 3 đến hoàn thiện ở Tuần 7), dự án RAG Knowledge Assistant đã đi qua đầy đủ vòng đời của một hệ thống GenAI thực tế: từ ý tưởng, thiết kế kiến trúc, xây dựng pipeline dữ liệu, tích hợp mô hình AI, mở API cho người dùng, đến giám sát vận hành và tự động đánh giá chất lượng. Bài học lớn nhất cá nhân tôi rút ra không nằm ở một service cụ thể nào, mà ở cách các thành phần serverless phối hợp với nhau qua sự kiện (event-driven) — và tầm quan trọng của việc đo lường bằng số liệu thay vì chỉ tin vào cảm giác "có vẻ ổn" khi làm việc với hệ thống AI.
