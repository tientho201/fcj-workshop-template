---
title: "Bản đề xuất"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# RAG Knowledge Assistant

## Giải pháp AWS Serverless cho hỏi đáp tài liệu nội bộ

### 1. Tóm tắt điều hành

RAG Knowledge Assistant là một chatbot hỏi đáp tài liệu nội bộ được xây dựng trên kiến trúc **RAG (Retrieval-Augmented Generation)**, cho phép nhân viên tải lên tài liệu (PDF, ảnh scan, văn bản thuần) và nhận câu trả lời được đối chiếu trực tiếp với nội dung đó, thay vì chỉ dựa vào kiến thức huấn luyện sẵn chung chung của LLM. Hệ thống chạy hoàn toàn trên các dịch vụ AWS Serverless — Lambda, SQS, Amazon Bedrock và DynamoDB (lưu cả vector và chỉ mục BM25, thay cho một search engine riêng) — kết hợp với Infrastructure as Code (Terraform) để đảm bảo hạ tầng triển khai nhất quán và có thể review qua Pull Request. Bên cạnh trải nghiệm hỏi đáp chính, nền tảng còn có cơ chế cache để kiểm soát chi phí gọi Bedrock, kiểm duyệt nội dung qua Bedrock Guardrails, giám sát vận hành thời gian thực, và vòng lặp đánh giá chất lượng tự động hàng ngày bằng framework RAGAS.

### 2. Tuyên bố vấn đề

_Vấn đề hiện tại_
Kiến thức của doanh nghiệp thường nằm rải rác trong các file PDF và tài liệu scan, khiến việc tra cứu thủ công chậm và lặp đi lặp lại. Các LLM có sẵn tuy trả lời trôi chảy nhưng không được đối chiếu với nội dung nội bộ thực tế, dễ dẫn đến câu trả lời sai nhưng nghe rất "tự tin" (hallucination). Ngoài ra, thường không có cách đo lường định lượng, có hệ thống để biết câu trả lời của chatbot có thực sự đáng tin hay không — các nhóm thường chỉ đánh giá dựa trên cảm tính.

_Giải pháp_
Nền tảng tiếp nhận tài liệu qua Amazon S3, đưa qua hàng đợi đệm Amazon SQS (kèm Dead Letter Queue để đảm bảo retry an toàn), và dùng AWS Lambda kết hợp Amazon Textract để số hóa các file scan. Nội dung trích xuất được chia nhỏ (chunk) và tạo embedding qua Amazon Bedrock, sau đó lưu trực tiếp trong Amazon DynamoDB dưới dạng vector đã đóng gói (packed) cùng dữ liệu tần suất từ (BM25) — một lớp hybrid search viết bằng Python tự xây (cosine similarity + BM25, hợp nhất qua Reciprocal Rank Fusion) chạy ngay trong Lambda, không cần tới một search engine riêng. Câu hỏi từ người dùng đi qua Amazon API Gateway (bảo vệ bởi Amazon Cognito), kiểm tra lớp cache ElastiCache Serverless trước để tránh gọi Bedrock lặp lại không cần thiết, truy xuất ngữ cảnh liên quan qua lớp hybrid search này, và sinh câu trả lời qua Amazon Bedrock (Claude 3 + Titan/Cohere Embeddings) được lọc qua Bedrock Guardrails để kiểm duyệt nội dung. Amazon CloudWatch, SNS và AWS Chatbot phân loại và định tuyến cảnh báo vận hành tới Slack theo mức độ nghiêm trọng, trong khi EventBridge Scheduler kích hoạt một Lambda chạy hàng ngày để đánh giá các chỉ số RAGAS (Faithfulness, Answer Relevancy, Context Precision) trên các hội thoại gần nhất.

_Lợi ích và hoàn vốn đầu tư (ROI)_
Giải pháp loại bỏ việc tra cứu tài liệu thủ công tốn thời gian bằng cách tập trung việc tìm kiếm vào một giao diện hội thoại duy nhất, trong khi cơ chế cache giúp giảm đáng kể chi phí gọi Bedrock lặp lại cho các câu hỏi có ngữ nghĩa tương tự nhau. Việc lưu vector và dữ liệu BM25 trực tiếp trong DynamoDB thay vì triển khai một search engine riêng (ví dụ OpenSearch Serverless) giúp loại bỏ một khoản chi phí nền chạy liên tục, giữ cho lớp retrieval tính phí theo mức sử dụng thực tế giống phần còn lại của hệ thống. Việc đánh giá tự động bằng RAGAS thay thế các nhận định cảm tính kiểu "có vẻ ổn" bằng các chỉ số định lượng, có thể theo dõi được, giúp phát hiện những lần suy giảm chất lượng câu trả lời (ví dụ độ trung thực/faithfulness thấp) mà việc review thủ công khó nhận ra. Vì toàn bộ hạ tầng là serverless và được triển khai qua Terraform, hệ thống có thể được phá hủy và dựng lại theo nhu cầu (qua script teardown/rebuild riêng) khi không sử dụng, giữ chi phí vận hành ở mức thấp chỉ khi hệ thống đang chạy, thay vì một mức chi phí cố định hàng tháng.

### 3. Kiến trúc giải pháp

Nền tảng được xây dựng quanh 4 luồng xử lý độc lập nhưng liên kết chặt chẽ với nhau, tất cả đều được triển khai qua Terraform để đảm bảo thay đổi hạ tầng nhất quán và có thể review được:

![RAG Knowledge Assistant System Architecture Overview](/images/5-Workshop/5.1-Workshop-overview/aws-new.drawio.png)

_Dịch vụ AWS sử dụng_

- **AWS Lambda**: Chạy logic xử lý tài liệu, chat engine và đánh giá RAGAS (Python 3.12).
- **Amazon S3**: Lưu trữ tài liệu gốc tải lên và kết quả đánh giá RAGAS.
- **Amazon SQS**: Đệm sự kiện xử lý tài liệu, kèm Dead Letter Queue để xử lý retry.
- **Amazon Textract**: Thực hiện OCR cho file scan và ảnh.
- **Amazon Bedrock**: Sinh embedding (Titan/Cohere) và câu trả lời (Claude 3), được kiểm soát qua Guardrails để kiểm duyệt nội dung.
- **Amazon DynamoDB**: Lưu chunk tài liệu (parent/child), vector đã đóng gói và dữ liệu BM25 phục vụ retrieval, lịch sử hội thoại và phản hồi (feedback) của người dùng — thay thế cho một search engine riêng phục vụ lập chỉ mục ngữ nghĩa.
- **Amazon API Gateway**: Cung cấp các endpoint chat, upload, và kiểm tra trạng thái.
- **Amazon Cognito**: Xác thực người dùng trước khi cấp quyền truy cập API.
- **Amazon ElastiCache Serverless**: Cache các cặp hỏi-đáp gần đây để giảm độ trễ và chi phí gọi Bedrock.
- **Amazon CloudWatch**: Thu thập log/metric, dashboard tùy chỉnh và cảnh báo (alarm).
- **Amazon SNS + AWS Chatbot**: Định tuyến cảnh báo theo mức độ nghiêm trọng tới Slack.
- **Amazon EventBridge Scheduler**: Kích hoạt job đánh giá RAGAS hàng ngày.
- **Terraform (HCP Terraform)**: Quản lý toàn bộ hạ tầng dưới dạng code, với remote state.

_Thiết kế thành phần_

- **Luồng 1 — Data Ingestion**: S3 nhận file tải lên, S3 Event kích hoạt SQS, và một Lambda (Document Processor) trích xuất văn bản (dùng Textract cho file scan), chia thành chunk cha/con (parent/child) và tạo embedding + dữ liệu BM25 lưu trực tiếp vào DynamoDB — không qua search engine ngoài.
- **Luồng 2 — Realtime Q&A**: API Gateway (sau lớp xác thực Cognito) gọi một Lambda Chat Engine kiểm tra cache, chạy hybrid search bằng Python tự viết (cosine similarity + BM25, hợp nhất qua Reciprocal Rank Fusion) trên DynamoDB để truy xuất ngữ cảnh, gọi Bedrock để sinh câu trả lời qua Guardrails, và ghi hội thoại vào DynamoDB.
- **Luồng 3 — Monitoring & Alert**: CloudWatch Alarms theo dõi lỗi Lambda, tỷ lệ lỗi 5xx của API, độ sâu DLQ, và tình trạng bị giới hạn (throttle) của Bedrock, publish tới các SNS topic phân theo mức độ nghiêm trọng và định tuyến tới Slack qua AWS Chatbot.
- **Luồng 4 — RAG Evaluation**: Một Lambda được lên lịch bởi EventBridge lấy mẫu các cặp hỏi-đáp gần đây mỗi ngày, chấm điểm bằng các chỉ số RAGAS, lưu kết quả vào S3 và publish điểm tổng hợp lên CloudWatch.

### 4. Triển khai kỹ thuật

_Các giai đoạn triển khai_
Dự án triển khai theo chu kỳ 5 tuần sau khi chốt đề tài, mỗi giai đoạn xây dựng trực tiếp trên nền của giai đoạn trước:

- **Nghiên cứu & thiết kế kiến trúc**: Chốt đề tài RAG Knowledge Assistant, đánh giá lựa chọn giữa Serverless tự xây và các dịch vụ managed sẵn có (ví dụ Bedrock Knowledge Bases), hoàn thiện proposal cùng sơ đồ kiến trúc tổng thể và luồng dữ liệu.
- **Chuẩn bị môi trường**: Chuẩn bị cấu trúc project Terraform/IaC, xin cấp quyền truy cập model trên Amazon Bedrock (Claude 3, Titan Embeddings), và cấu hình môi trường phát triển.
- **Xây dựng các luồng cốt lõi**: Triển khai Luồng 1 (Data Ingestion) trước, sau đó tới Luồng 2 (Realtime Q&A kèm Semantic Cache), kiểm thử thực tế từng luồng trước khi chuyển sang luồng tiếp theo.
- **Khả năng quan sát & chất lượng**: Triển khai Luồng 3 (Monitoring & Alerting) và Luồng 4 (đánh giá RAGAS) để hệ thống có thể tự phát hiện sự cố và tự đo lường chất lượng câu trả lời.
- **Hoàn thiện & bàn giao**: Tinh chỉnh tham số retrieval dựa trên kết quả RAGAS, kiểm thử tải (load test), rà soát quyền IAM, tái cấu trúc Terraform theo module, hoàn thiện tài liệu và trình bày demo trực tiếp.

_Yêu cầu kỹ thuật_

- **Tài khoản & vùng (region)**: Tài khoản AWS ở vùng **us-east-1 (N. Virginia)** — vùng hỗ trợ đầy đủ các model Bedrock cần dùng, cùng với tài khoản HCP Terraform để quản lý remote state.
- **Công cụ**: Terraform 1.5+, AWS CLI v2, Python 3.12 (runtime cho Lambda), Git, và một code editor.
- **Quyền hạn**: Một IAM policy triển khai được giới hạn đúng phạm vi các nhóm dịch vụ sử dụng (Lambda, S3, SQS/SNS/EventBridge, DynamoDB/ElastiCache, Bedrock/Textract, API Gateway/Cognito, CloudWatch/Chatbot, quản lý IAM role) — tuân thủ nguyên tắc least privilege xuyên suốt.
- **CI/CD**: GitHub Actions chạy kiểm tra/plan trên mọi Pull Request, trong khi `terraform apply` chỉ được kích hoạt thủ công và có review, thay vì tự động chạy khi merge.

### 5. Lộ trình & Mốc triển khai

_Lộ trình dự án_ (5 tuần, sau 2 tuần học nền tảng AWS)

- **Tuần 1**: Lên ý tưởng đề tài, chốt proposal RAG Knowledge Assistant, thiết kế sơ đồ kiến trúc, chuẩn bị môi trường phát triển.
- **Tuần 2**: Hoàn thành Luồng 1 — Data Ingestion từ đầu đến cuối (S3 → SQS → Lambda → OCR → embedding).
- **Tuần 3**: Hoàn thành Luồng 2 — Realtime Q&A với API có xác thực, cache và Guardrails.
- **Tuần 4**: Hoàn thành Luồng 3 — Monitoring & Alerting, và Luồng 4 — đánh giá RAGAS tự động.
- **Tuần 5**: Tinh chỉnh chất lượng retrieval, kiểm thử tải, rà soát bảo mật IAM, tái cấu trúc IaC, hoàn thiện tài liệu và demo trực tiếp.

### 6. Ước tính ngân sách

Nhờ lưu vector và dữ liệu BM25 trong DynamoDB (tính phí theo request) thay vì triển khai một search engine riêng, yếu tố chi phí nền chạy liên tục chính chỉ còn là dung lượng tối thiểu được cấp phát của Amazon ElastiCache Serverless, cộng thêm chi phí theo lượt gọi Bedrock.

_Chi phí hạ tầng_

- **Chi phí vận hành ước tính**: khoảng 2,5 USD/ngày khi hệ thống đang được triển khai (dung lượng nền của ElastiCache Serverless là yếu tố chi phí chính, trong khi Lambda, S3, SQS, DynamoDB và API Gateway tính phí theo lượt sử dụng và chiếm tỷ trọng nhỏ hơn nhiều ở quy mô này).
- **Các biện pháp kiểm soát chi phí**:
  - Dùng DynamoDB thay vì một search engine riêng để lưu vector/BM25 giúp tránh thêm một khoản chi phí nền chạy liên tục cho lớp retrieval.
  - Cơ chế cache exact-match giúp giảm chi phí gọi Bedrock lặp lại cho các câu hỏi giống hệt nhau.
  - Script teardown/rebuild riêng cho phép phá hủy toàn bộ hạ tầng (kèm sao lưu tài liệu) khi không sử dụng, tránh phát sinh chi phí cố định hàng tháng.
  - Một AWS Budget alert giám sát chi tiêu hàng tháng, hoạt động độc lập với vòng đời của hạ tầng chính.

### 7. Đánh giá rủi ro

_Ma trận rủi ro_

- Chậm trễ trong việc được cấp quyền truy cập model trên Amazon Bedrock: Ảnh hưởng trung bình, xác suất trung bình.
- Chất lượng retrieval/câu trả lời thấp (hallucination, ngữ cảnh truy xuất không khớp): Ảnh hưởng cao, xác suất trung bình.
- Vượt ngân sách do dịch vụ chạy nền liên tục (ElastiCache Serverless): Ảnh hưởng trung bình, xác suất thấp.
- Cấu hình sai quyền IAM giữa các dịch vụ: Ảnh hưởng cao, xác suất thấp.
- Giới hạn concurrency của Lambda khi tải cao: Ảnh hưởng trung bình, xác suất thấp.

_Chiến lược giảm thiểu_

- Bedrock: Gửi yêu cầu cấp quyền truy cập model càng sớm càng tốt ngay trong giai đoạn chuẩn bị môi trường, trước khi việc phát triển phụ thuộc vào nó.
- Chất lượng retrieval: Xây dựng vòng lặp đánh giá RAGAS sớm để phát hiện suy giảm chất lượng bằng số liệu thay vì phỏng đoán, làm cơ sở để tinh chỉnh cụ thể (kích thước chunk, trọng số hybrid search).
- Chi phí: Cảnh báo AWS Budget kết hợp với việc phá hủy hạ tầng bằng script khi không sử dụng.
- IAM: Áp dụng chính sách least-privilege ngay từ đầu và thực hiện một đợt rà soát quyền hạn riêng trước khi bàn giao.
- Concurrency: Kiểm thử tải trước buổi demo cuối để phát hiện giới hạn sớm và ghi chú phương án mở rộng.

_Kế hoạch dự phòng_

- Nếu việc cấp quyền Bedrock bị chậm, tiếp tục phát triển hạ tầng và pipeline bằng lệnh gọi embedding/generation giả lập (mock), rồi tích hợp lệnh gọi model thật khi được cấp quyền.
- Nếu chi phí tiệm cận ngưỡng ngân sách, chạy ngay script teardown để phá hủy các tài nguyên không thiết yếu.
- Nếu chất lượng retrieval không thể cải thiện đủ trong thời gian cho phép, ghi nhận rõ ràng khoảng cách này và đưa vào danh sách việc cần làm tiếp theo, thay vì âm thầm bàn giao một hệ thống chất lượng chưa đạt.

### 8. Kết quả kỳ vọng

_Cải tiến kỹ thuật_
Tự động hóa việc tiếp nhận tài liệu và OCR thay thế xử lý thủ công. Câu trả lời được cache trả về trong chưa tới một giây cho các câu hỏi lặp lại, so với vài giây cho một lượt gọi Bedrock mới. Đánh giá RAGAS tự động thay thế việc kiểm tra chất lượng cảm tính bằng chấm điểm định lượng hàng ngày. Cảnh báo thời gian thực, phân loại theo mức độ nghiêm trọng, rút ngắn thời gian phát hiện sự cố vận hành.

_Giá trị dài hạn_
Một kiến trúc tham chiếu có thể tái sử dụng, có tài liệu đầy đủ cho các hệ thống GenAI Serverless trên AWS, kinh nghiệm thực tế với Infrastructure as Code (Terraform) và thiết kế theo hướng sự kiện (event-driven), cùng nền tảng có thể mở rộng cho các bài toán quản lý tri thức doanh nghiệp rộng hơn trong tương lai.
