---
title: "Worklog Tuần 8"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.8. </b> "
---

### Mục tiêu tuần 8:

- Rà soát và tổng hợp lại toàn bộ hành trình 7 tuần: kiến thức nền tảng AWS (Tuần 1-2) và dự án RAG Knowledge Assistant (Tuần 3-7).
- Kiểm tra lại xem toàn bộ hệ thống có còn chạy đúng từ đầu đến cuối hay không, đối chiếu kết quả cuối cùng với đề xuất ban đầu ở Tuần 3.
- Tự đánh giá kiến thức đã tích lũy so với mục tiêu học tập ban đầu, xác định điểm mạnh và những gì còn thiếu.
- Tổng hợp lại worklog và hoàn thiện tài liệu báo cáo thực tập.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                                                                                                            | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu     |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ---------------- | -------------------- |
| 2   | **Rà soát kiến thức AWS nền tảng (Tuần 1-2):**<br>- Tự kiểm tra lại các dịch vụ cốt lõi đã học: IAM, VPC, EC2, S3, RDS, Lightsail, Auto Scaling, CloudWatch, Route 53, DynamoDB, ElastiCache, CloudFront.<br>- Thử tự giải thích lại vài kiến trúc (ví dụ: mô hình VPC/Multi-AZ) mà không xem lại note, để chắc chắn là hiểu thật chứ không chỉ quen mặt.                    | 10/08/2026   | 10/08/2026       | Ghi chú cá nhân       |
| 3   | **Rà soát toàn bộ project từ đầu đến cuối:**<br>- Đi lại từng luồng của RAG Knowledge Assistant (Ingestion, Realtime QA, Monitoring, Evaluation).<br>- Test lại hệ thống thật: upload một tài liệu mới, hỏi đáp thử, kiểm tra Semantic Cache có hit đúng không, xác nhận job đánh giá RAGAS vẫn chạy đúng lịch.                  | 11/08/2026   | 11/08/2026       | Dự án cá nhân        |
| 4   | **Đối chiếu kết quả với đề xuất ban đầu:**<br>- So sánh những gì thực sự đã làm được với bản proposal và sơ đồ kiến trúc ở Tuần 3.<br>- Liệt kê các hạng mục đã hoàn thành, hạng mục làm được một phần, và hạng mục còn để ngỏ (ví dụ: rate-limiting cho API, mở rộng bộ dữ liệu đánh giá — đúng như góp ý nhận được ở Tuần 7). | 12/08/2026   | 12/08/2026       | Dự án cá nhân        |
| 5   | **Hoàn thiện tài liệu:**<br>- Gom lại toàn bộ worklog 8 tuần thành một bản báo cáo thực tập mạch lạc.<br>- Cập nhật lại runbook kiến trúc và README theo những thay đổi phát sinh trong quá trình rà soát.<br>- Viết phần "bài học rút ra", bao gồm cả kiến thức AWS nền tảng lẫn kiến trúc Serverless/GenAI.                    | 13/08/2026   | 13/08/2026       | Dự án cá nhân        |
| 6   | **Tổng kết & khép lại tuần:**<br>- Trình bày phần rà soát tổng thể trước nhóm/mentor: đã học được gì, project đi từ đề xuất đến trạng thái sẵn sàng vận hành như thế nào, và nếu làm lại thì sẽ thay đổi điều gì.<br>- Ghi nhận góp ý cuối cùng và phác thảo các hướng phát triển tiếp theo sau kỳ thực tập.                    | 14/08/2026   | 14/08/2026       | Dự án cá nhân        |

### Kết quả đạt được tuần 8:

- **Xác nhận vẫn nhớ vững kiến thức AWS nền tảng:** Rà lại nội dung Tuần 1-2 mà không cần mở note ra xem, tôi nhận thấy mình vẫn nắm chắc các nhóm dịch vụ cốt lõi (Compute, Storage, Networking, Database) — và quan trọng hơn là hiểu được cách chúng kết hợp với nhau thành một kiến trúc hoàn chỉnh. Chính khả năng "ghép nối" này là nền tảng giúp tôi triển khai được project ở Tuần 3-7.

- **Xác nhận project vẫn chạy ổn định từ đầu đến cuối:** Chạy lại toàn bộ luồng (upload tài liệu → OCR → hỏi đáp → cache → giám sát → đánh giá RAGAS) cho thấy hệ thống ổn định lâu dài chứ không chỉ chạy đúng một lần cho buổi demo.

- **Đối chiếu thẳng thắn với đề xuất ban đầu:** So với proposal ở Tuần 3, cả 4 luồng chính đều đã hoàn thành đầy đủ. Các hạng mục phụ (rate-limiting cho API, mở rộng bộ dữ liệu đánh giá, khả năng scale ngang vượt giới hạn concurrency hiện tại của Lambda) được ghi nhận rõ ràng như những bước tiếp theo, chứ không bị "lặng lẽ" bỏ qua.

- **Tài liệu dự án đã được tổng hợp đầy đủ:** Runbook kiến trúc, README triển khai và worklog 8 tuần giờ đã thống nhất và cập nhật, đủ để người khác có thể tiếp nhận lại project mà không cần phụ thuộc vào việc hỏi lại tôi.

- **Nhìn lại toàn bộ hành trình:** Từ việc tạo tài khoản AWS ở Tuần 1 cho đến một hệ thống GenAI tự động, tự giám sát và tự đánh giá chất lượng ở Tuần 7, bài học lớn nhất không nằm ở một dịch vụ AWS cụ thể nào, mà ở sự chuyển dịch tư duy từ "làm cho chạy được" sang "làm cho vận hành được": thiết kế theo hướng sự kiện (event-driven), phân quyền tối thiểu (least-privilege IAM), khả năng quan sát hệ thống (observability), và cải tiến dựa trên số liệu (RAGAS) — tất cả đều hướng về cùng một triết lý đó.
