---
title: "Cơ chế Resume OCR và xử lý lỗi"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.3.5 </b> "
---

Trang này trình bày 2 cơ chế đặc biệt của Lambda `document_processor`: lối vào thứ hai để tiếp tục/hủy tài liệu đang chờ xác nhận OCR, và cách xử lý khi gặp lỗi runtime bất ngờ.

#### Lối vào thứ hai: Resume / Cancel OCR

Lambda này có **2 lối vào**:

- **Lối chính qua SQS** (`event["Records"]`): Xử lý bất đồng bộ các file mới upload lên S3.
- **Lối thứ hai không qua SQS** (gọi trực tiếp khi `event` có key `"action"`): Nhận lệnh trực tiếp từ `chat_engine` (endpoint `/documents-decision`) để tiếp tục (resume) hoặc hủy (cancel) tài liệu PDF scan đang tạm dừng ở trạng thái `awaiting_ocr_confirmation` (xem trang [5.3.3 - Trích xuất văn bản theo loại file](../5.3.3-Text-Extraction/)).

```python
def lambda_handler(event, context):
    # Lối vào thứ hai: Direct invoke từ chat-engine (luôn mang key "action")
    if "action" in event:
        return _handle_direct_invoke(event)

    batch_item_failures = []

    # Lối vào chính: Xử lý danh sách record từ SQS
    for record in event.get("Records", []):
        try:
            body = json.loads(record["body"])
            if "Records" not in body:
                continue

            for s3_event in body["Records"]:
                bucket = s3_event["s3"]["bucket"]["name"]
                key = urllib.parse.unquote_plus(s3_event["s3"]["object"]["key"])
                try:
                    _process_s3_object(bucket, key)
                except Exception as error:
                    _write_status(
                        f"{bucket}/{key}", key, "failed", Tracer(), error=str(error)[:500]
                    )
                    raise

        except Exception as error:
            logger.exception("Failed to process SQS record %s", record.get("messageId"))
            _report_to_function_dlq(record, error)
            batch_item_failures.append({"itemIdentifier": record["messageId"]})

    return {"batchItemFailures": batch_item_failures}
```

Hàm `_handle_direct_invoke` điều hướng xử lý theo `action`:

```python
def _handle_direct_invoke(event):
    action = event["action"]
    bucket = event["bucket"]
    key = event["key"]
    document_id = f"{bucket}/{key}"

    if action == "resume_ocr":
        try:
            # force_ocr=True: Bỏ qua đọc pypdf, gọi trực tiếp Textract OCR
            _process_s3_object(bucket, key, force_ocr=True)
            return {"ok": True}
        except Exception as error:
            logger.exception("Forced-OCR resume failed for %s", document_id)
            _write_status(document_id, key, "failed", Tracer(), error=str(error)[:500])
            return {"ok": False, "error": str(error)[:500]}

    if action == "cancel":
        _write_status(document_id, key, "cancelled", Tracer())
        return {"ok": True}

    raise ValueError(f"Unknown action '{action}'")
```

Phía `chat-engine` thực hiện gọi bất đồng bộ sang Lambda `document_processor` khi nhận yêu cầu từ người dùng tại endpoint `/documents-decision`:

```python
# Trích từ modules/query/lambda_src/chat_engine/handler.py
lambda_client.invoke(
    FunctionName=DOCUMENT_PROCESSOR_FUNCTION_NAME,
    InvocationType="Event", # Gọi bất đồng bộ (fire-and-forget)
    Payload=json.dumps({"action": action, "bucket": bucket, "key": key}).encode("utf-8"),
)
```

Cơ chế này cho phép người dùng, thông qua giao diện chat (endpoint `/documents-decision`), chủ động quyết định có chấp nhận chi phí OCR cho một file PDF scan hay không, thay vì hệ thống tự động OCR mọi trường hợp.

#### Xử lý lỗi runtime bất ngờ (Function DLQ)

Khác với `ingestion_dlq` (SQS tự động đẩy vào sau 3 lần retry — xem trang [5.3.1 - Hạ tầng: S3 và SQS](../5.3.1-Infrastructure-S3-SQS/)), queue `document_processor_fn_dlq` được chính code chủ động ghi khi gặp lỗi không lường trước (bug trong code, không phải do file hỏng):

```python
def _report_to_function_dlq(record, error):
    if not FUNCTION_DLQ_URL:
        return
    try:
        sqs.send_message(
            QueueUrl=FUNCTION_DLQ_URL,
            MessageBody=json.dumps(
                {
                    "error": str(error),
                    "traceback": traceback.format_exc(),
                    "original_record": record,
                }
            ),
        )
    except Exception:
        logger.exception("Failed to write to the function-level DLQ")
```

Tách 2 tầng DLQ này giúp đội vận hành phân biệt nhanh khi rà soát: nếu message nằm ở `ingestion_dlq` → nên kiểm tra lại tệp nguồn; nếu nằm ở `document_processor_fn_dlq` → nên xem traceback chi tiết để sửa lỗi mã nguồn.

---

#### Nội dung tiếp theo

Tiếp theo: [5.3.6 - Kiểm thử End-to-End](../5.3.6-End-To-End-Testing/)
