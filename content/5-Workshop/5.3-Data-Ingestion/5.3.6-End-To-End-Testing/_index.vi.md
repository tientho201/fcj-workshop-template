---
title: "Kiểm thử End-to-End"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.3.6 </b> "
---

Sau khi hoàn tất hạ tầng ([5.3.1](../5.3.1-Infrastructure-S3-SQS/), [5.3.2](../5.3.2-Infrastructure-DynamoDB-IAM/)) và logic xử lý ([5.3.3](../5.3.3-Text-Extraction/), [5.3.4](../5.3.4-Chunking-Embedding/), [5.3.5](../5.3.5-Resume-OCR-Error-Handling/)), bước cuối cùng là kiểm thử toàn bộ luồng end-to-end.

#### Kịch bản test

Upload lần lượt để kiểm tra từng nhánh xử lý:

1. **File PDF text thuần** → `pypdf` đọc được ngay, không cần OCR.
2. **File PDF dạng scan** → status chuyển `awaiting_ocr_confirmation`, gọi `/documents-decision` xác nhận OCR, sau đó Textract xử lý (xem lại [5.3.3](../5.3.3-Text-Extraction/) và [5.3.5](../5.3.5-Resume-OCR-Error-Handling/)).
3. **File ảnh (.png/.jpg)** → luôn OCR ngay, không hỏi.
4. **File .txt/.md** → decode UTF-8 trực tiếp.

Sau mỗi lần, kiểm tra theo đúng thứ tự:

**SQS** (message đã tiêu thụ) → **CloudWatch Logs** (không lỗi, hoặc thấy log `_report_to_function_dlq` nếu có bug) → **bảng `ingestion_status`** (status cuối = `completed`) → **bảng `parent_chunks`/`child_chunks`** (đã có item mới).

##### Hình 1: Kết quả test end-to-end với file pdf

![Kết quả test end-to-end với file pdf](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-1.png)

##### Hình 2: Kết quả test end-to-end với file txt

- Với file txt thì tài liệu sẽ được trích xuất trong hộp thoại để có thể bấm "Tài lên & xử lý"

![Kết quả test end-to-end với file txt](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-2.png)

##### Hình 3: Kết quả test end-to-end với file png

- Với file png thì sẽ gửi nguyên dạng (base64) để document-processor chạy Textract OCR — không hiển thị được dạng text ở hộp thoại upload
  ![Kết quả test end-to-end với file png](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-3.png)

##### Hình 4: Kết quả test end-to-end với file md

- Với file md thì tài liệu sẽ được trích xuất trong hộp thoại để có thể bấm "Tài lên & xử lý"

![Kết quả test end-to-end với file md](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-4.png)

#### Kết quả đạt được

- Pipeline ingest hoạt động event-driven hoàn toàn qua S3 → SQS, không cần Lambda trung gian nhận event.
- Phân biệt rõ 2 loại lỗi (file xấu vs. code bug) qua 2 tầng DLQ riêng biệt, giúp debug nhanh hơn.
- Xử lý được cả 3 loại tài liệu (text thuần, PDF có/không có lớp text, ảnh scan) với luồng xác nhận OCR chủ động thay vì OCR mù mọi PDF (tiết kiệm chi phí Textract).
- Toàn bộ vector và chỉ số BM25 lưu gọn trong DynamoDB (dạng Binary + JSON string), loại bỏ hoàn toàn nhu cầu vận hành thêm một cụm OpenSearch Serverless riêng.
- IAM Role tuân thủ least-privilege, có ghi chú rõ ngoại lệ (Textract) để tránh hiểu nhầm khi review bảo mật.
