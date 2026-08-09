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

![Kết quả test end-to-end với cả 4 loại file](../images/11-end-to-end-test-result.png)
*Ảnh minh họa: bảng `ingestion_status` với 4 document ở trạng thái `completed`, và `child_chunks` tăng số lượng item tương ứng.*

#### Kết quả đạt được

- Pipeline ingest hoạt động event-driven hoàn toàn qua S3 → SQS, không cần Lambda trung gian nhận event.
- Phân biệt rõ 2 loại lỗi (file xấu vs. code bug) qua 2 tầng DLQ riêng biệt, giúp debug nhanh hơn.
- Xử lý được cả 3 loại tài liệu (text thuần, PDF có/không có lớp text, ảnh scan) với luồng xác nhận OCR chủ động thay vì OCR mù mọi PDF (tiết kiệm chi phí Textract).
- Toàn bộ vector và chỉ số BM25 lưu gọn trong DynamoDB (dạng Binary + JSON string), loại bỏ hoàn toàn nhu cầu vận hành thêm một cụm OpenSearch Serverless riêng.
- IAM Role tuân thủ least-privilege, có ghi chú rõ ngoại lệ (Textract) để tránh hiểu nhầm khi review bảo mật.

#### Checklist hoàn thành Luồng 1

- [ ] S3 bucket `raw_documents` đã bật versioning, SSE-S3, lifecycle Glacier 90 ngày
- [ ] SQS `ingestion_queue` + 2 DLQ (`ingestion_dlq`, `document_processor_fn_dlq`) hoạt động đúng, `batch_size = 1`
- [ ] 3 bảng DynamoDB (`parent_chunks`, `child_chunks`, `ingestion_status`) đã tạo
- [ ] IAM Role `document_processor` chỉ có đúng 4 nhóm quyền least-privilege
- [ ] Nhánh xử lý PDF/ảnh/text đều hoạt động đúng, luồng `awaiting_ocr_confirmation` test được qua `/documents-decision`
- [ ] Lối vào `resume_ocr`/`cancel` gọi trực tiếp từ `chat_engine` hoạt động đúng
- [ ] Test end-to-end cả 4 loại file, dữ liệu xuất hiện đúng trong DynamoDB

{{% notice warning %}}
Vì Luồng 1 không còn dùng OpenSearch, **Luồng 2 (Hỏi đáp Realtime)** cũng cần cập nhật lại: thay vì query OpenSearch, sẽ đọc trực tiếp `child_chunks` từ DynamoDB, tính cosine similarity với vector câu hỏi + BM25 score, rồi kết hợp bằng RRF.
{{% /notice %}}
