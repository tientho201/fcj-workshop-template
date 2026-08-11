---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# Từ lý thuyết đến thực thi: Bài toán Data Engineering trong một đồ án GenAI/RAG thực tế

Xin chào mọi người,

Ở bài viết trước, mình đã chia sẻ góc nhìn về việc một Data Engineer nên ưu tiên học những gì trên AWS để tối ưu hóa thời gian và sẵn sàng cho công việc.

Hôm nay, mình muốn chia sẻ kĩ hơn về cách mình đem những tư duy và dịch vụ AWS đó áp dụng trực tiếp vào đồ án thực tế của mình. Mặc dù bài toán chính của đồ án thuộc mảng GenAI / RAG (Retrieval-Augmented Generation), nhưng khi bắt tay vào triển khai kiến trúc hệ thống, mình nhận ra: Có tới 80% sức mạnh và độ ổn định của hệ thống nằm ở khâu Data Engineering.

Dưới đây là bức tranh toàn cảnh về cách mình ứng dụng các kỹ thuật xử lý dữ liệu cloud trong đồ án này.

## 1. Data Ingestion & Event-Driven Pipeline (Luồng xử lý nạp dữ liệu)

Khi xử lý tài liệu đầu vào, nếu chỉ làm theo cách truyền thống là gọi hàm xử lý trực tiếp thì hệ thống rất dễ bị nghẽn (bottleneck) hoặc quá tải. Do đó, mình xây dựng theo hướng Event-Driven Architecture:

- **Tự động kích hoạt (S3 Event Trigger):** Mỗi khi người dùng hoặc Admin upload tài liệu mới (PDF/TXT/Scan) vào Amazon S3 Raw Documents, một sự kiện (S3 Event) sẽ ngay lập tức kích hoạt luồng xử lý mà không cần polling thủ công.
- **Hàng đợi đệm & Phân rã hệ thống (Amazon SQS):** Để hệ thống không bị ngợp khi lượng file đẩy vào quá lớn cùng lúc, mình đưa dữ liệu qua Amazon S3 Event → Amazon SQS (Buffer Queue). Việc sử dụng SQS đóng vai trò làm điểm tựa chịu tải, kết hợp cơ chế Retry tự động và Dead Letter Queue (DLQ) giúp hứng các message lỗi, đảm bảo tuyệt đối không bị mất mát dữ liệu (zero data loss).

## 2. ETL & Unstructured Data Processing (Trích xuất & Biến đổi dữ liệu)

Dữ liệu văn bản phi cấu trúc (Unstructured Data) cần trải qua một quy trình ETL nghiêm ngặt trước khi có thể phục vụ cho các mô hình AI:

- **Trích xuất dữ liệu (OCR):** AWS Lambda sẽ nhận tin nhắn từ SQS, tự động gọi Amazon Textract để thực hiện OCR, trích xuất chính xác văn bản từ các file Scan hay PDF phức tạp.
- **Cấu trúc hóa & Lưu trữ Vector (Chunking & Vectorization):** Dữ liệu sau khi xử lý sẽ được cắt nhỏ (chunking) theo mô hình Parent-Child, sau đó gọi Embedding API để biến đổi thành các vector. Toàn bộ thông tin cấu trúc này được lưu trữ vào Amazon DynamoDB – tối ưu hóa cho các truy vấn Hybrid Search (kết hợp Cosine Similarity và BM25 bằng thuật toán RRF) nhằm đạt độ chính xác cao nhất khi tìm kiếm.

## 3. Low-Latency Data Retrieval & Caching (Tối ưu hóa truy vấn thời gian thực)

Một bài toán cốt lõi của Data Engineering trong các ứng dụng Web/App là độ trễ (latency) và chi phí (cost):

- **Semantic Caching:** Mình tích hợp Amazon ElastiCache Serverless làm lớp Caching thông minh cho câu hỏi. Nếu câu hỏi mới có ngữ nghĩa tương tự câu hỏi cũ, hệ thống sẽ trả về kết quả ngay trong Cache thay vì phải gọi lại mô hình LLM đắt đỏ. Kỹ thuật này giúp giảm tối đa độ trễ cho người dùng cuối (Real-time serving).
- **Quản lý trạng thái & Phản hồi (Transaction Store):** Toàn bộ lịch sử hội thoại (Chat History) và dữ liệu phản hồi (Feedback Store) được lưu trữ trên DynamoDB – dòng NoSQL Database đáp ứng tốc độ ghi siêu nhanh với độ trễ chỉ tính bằng milisecond.

## 4. Automated Batch Processing & Continuous Evaluation (Pipeline đánh giá tự động)

Không dừng lại ở luồng dữ liệu thời gian thực, một hệ thống dữ liệu chuẩn mực luôn cần luồng xử lý định kỳ (Batch Pipeline) để đánh giá chất lượng:

- **Tự động hóa Batch Job:** Mình dùng Amazon EventBridge Scheduler kết hợp AWS Lambda để khởi chạy pipeline đánh giá chất lượng mô hình (theo bộ tiêu chí RAGAS) hoàn toàn tự động theo khung giờ cố định hàng ngày.
- **Lưu trữ Data Lake định kỳ:** Kết quả đánh giá từng ngày được đẩy về lại Amazon S3 Evaluation Results đóng vai trò như một Data Lake lưu trữ log lịch sử. Điều này giúp team dễ dàng theo dõi, phân tích xu hướng biến động chất lượng của hệ thống theo thời gian.

## 5. Data Observability & Monitoring (Giám sát dòng chảy dữ liệu)

Cuối cùng, dữ liệu chạy trong hệ thống Cloud cần phải "nhìn thấy được" (observable) để kịp thời phát hiện sự cố:

- **Tập trung hóa Log & Metric:** Dùng Amazon CloudWatch để thu thập log, theo dõi các chỉ số quan trọng của pipeline như: độ sâu hàng đợi DLQ (DLQ Depth), tỉ lệ lỗi 5xx của API Gateway, cho đến các Custom Metrics về AI như Faithfulness, Relevancy, Precision.
- **Cảnh báo thông minh (Alert Routing):** Phân loại sự cố theo cấp độ (Warning vs Critical). Khi xảy ra lỗi nghiêm trọng (Critical), Amazon SNS sẽ phối hợp cùng AWS Chatbot để đẩy ngay thông báo cảnh báo về Slack hoặc PagerDuty cho đội ngũ kỹ thuật xử lý.

## Ba "điểm sáng" Data Engineering mình rút ra từ đồ án

Nếu bạn cũng đang chuẩn bị làm đồ án hoặc project cá nhân, đây là 3 tư duy Data Engineering mà mình thấy đáng giá nhất khi đưa vào bài toán:

- **Tính Decoupled (Tách biệt hệ thống):** Khâu nạp dữ liệu (Ingestion) và khâu truy vấn (Serving) hoạt động độc lập qua hàng đợi SQS và DynamoDB. Dù Admin có upload hàng ngàn file PDF cùng lúc thì trải nghiệm chat của người dùng cuối vẫn hoàn toàn mượt mà, không hề bị ảnh hưởng.
- **Xử lý bất đồng bộ (Asynchronous Serverless):** Việc kết hợp S3 Event + SQS + Lambda giúp hệ thống hoạt động bất đồng bộ hoàn toàn. Tài nguyên máy tính chỉ bật lên khi có dữ liệu chạy qua, giúp tối ưu hóa 100% chi phí vận hành Cloud.
- **Tối ưu hóa truy vấn dữ liệu (Query Optimization):** Thay vì truy vấn "ngây thơ" thẳng vào LLM, việc kết hợp Semantic Cache (ElastiCache) và Hybrid Search (BM25 + Vector Search) chính là minh chứng cho tư duy tối ưu hiệu năng dữ liệu của một Data Engineer.

## Lời kết

Đồ án này đã giúp mình chứng minh một điều: Làm Data Engineer không chỉ là viết các câu lệnh SQL hay chạy script Spark, mà là khả năng thiết kế một hạ tầng Cloud tin cậy, tự động hóa luồng dữ liệu và tối ưu chi phí vận hành cho hệ thống.

Hy vọng góc nhìn chia sẻ từ đồ án thực tế này sẽ cho các bạn thấy một bức tranh sinh động hơn về cách các dịch vụ AWS như S3, SQS, Lambda, DynamoDB, ElastiCache phối hợp với nhau trong công việc.
##Link
<https://www.facebook.com/groups/awsstudygroupfcj/permalink/2240430060055287/#>
