---
title: "Xử lý lỗi và quyết định OCR"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.4.6 </b> "
---

Ngoài route chính `_handle_chat` đã trình bày ở 3 trang trước, Lambda `chat_engine` còn xử lý lỗi đặc biệt từ Bedrock, và có 1 route phụ kết nối ngược lại Luồng 1.

#### Xử lý riêng lỗi Throttling từ Bedrock

```python
from botocore.exceptions import ClientError

def _handle_chat(event):
    try:
        ...  # toàn bộ pipeline: cache, rewrite, guardrail, retrieval, generation
    except ClientError as error:
        code = error.response.get("Error", {}).get("Code", "")
        if code in ("ThrottlingException", "TooManyRequestsException"):
            logger.exception("Bedrock throttled the request")
            return _response(
                429,
                {
                    "error": "Hệ thống đang quá tải (Bedrock throttling). "
                    "Vui lòng thử lại sau ít giây.",
                    "retryable": True,
                },
            )
        logger.exception("chat-engine failed (AWS error %s)", code)
        return _response(500, {"error": "Internal error handling the request."})

    except Exception:
        logger.exception("chat-engine failed")
        return _response(500, {"error": "Internal error handling the request."})

    finally:
        semantic_cache.close()
```

{{% notice note %}}
Thay vì để lỗi Throttling từ Bedrock lẫn vào nhóm lỗi 500 chung chung, code **bắt riêng** `ThrottlingException`/`TooManyRequestsException` và trả về **HTTP 429** kèm `{"retryable": true}` — giúp client (frontend) biết chính xác nên tự động thử lại sau một khoảng thời gian, thay vì hiển thị lỗi hệ thống cho người dùng. Đồng thời log ở mức **ERROR** để **CloudWatch Metric Filter ở Luồng 3** bắt được và kích hoạt alarm `CW Alarm Bedrock Throttle` (đã cấu hình ở phần Monitoring).
{{% /notice %}}

#### Route phụ: `/documents-decision` — kết nối sang Luồng 1

```python
def _handle_ocr_decision(event):
    """POST /documents-decision — a human resolving the pause
    document-processor leaves behind for a PDF with no embedded text layer
    (status `awaiting_ocr_confirmation`; see that Lambda's
    _process_s3_object). Body: {"document_id": "...", "decision": "ocr" |
    "cancel"}.

    Invokes document-processor asynchronously (InvocationType="Event"):
    Textract OCR can run longer than API Gateway's 29s integration limit,
    and the UI is already polling GET /status for progress, so there is
    nothing to wait for synchronously here.
    """
    body = json.loads(event.get("body") or "{}")
    document_id = (body.get("document_id") or "").strip()
    decision = body.get("decision")

    if not document_id or "/" not in document_id:
        return _response(400, {"error": "Cần có 'document_id' hợp lệ."})
    if decision not in ("ocr", "cancel"):
        return _response(400, {"error": "'decision' phải là 'ocr' hoặc 'cancel'."})

    # document_id is always f"{bucket}/{key}" (see _handle_upload) and keys
    # in this bucket are kept flat, but partition (not split) is used anyway
    # so a key that somehow contains "/" still reassembles correctly instead
    # of being truncated.
    bucket, _, key = document_id.partition("/")
    action = "resume_ocr" if decision == "ocr" else "cancel"

    lambda_client.invoke(
        FunctionName=DOCUMENT_PROCESSOR_FUNCTION_NAME,
        InvocationType="Event",
        Payload=json.dumps({"action": action, "bucket": bucket, "key": key}).encode("utf-8"),
    )

    return _response(
        202,
        {
            "document_id": document_id,
            "decision": decision,
            "message": "Đã ghi nhận quyết định, đang xử lý bất đồng bộ.",
        },
    )

```

Hàm `_handle_ocr_decision` (`handler.py:369-410`) gọi `lambda_client.invoke(..., InvocationType="Event")` sang **`document_processor` (Lambda của Luồng 1) một cách bất đồng bộ** — đây chính là điểm nối giữa 2 luồng cho cơ chế xác nhận OCR thủ công đã trình bày ở [trang 5.3.5](../../5.3-Data-Ingestion/5.3.5-Resume-OCR-Error-Handling/).

{{% notice note %}}
Việc gọi `InvocationType="Event"` (bất đồng bộ) thay vì `RequestResponse` (đồng bộ) rất quan trọng: người dùng bấm "Xác nhận OCR" trên giao diện chỉ cần nhận phản hồi ngay (202 Accepted) mà không phải chờ toàn bộ quá trình OCR + embedding + lưu DynamoDB chạy xong — quá trình đó có thể mất vài giây đến vài chục giây tùy độ dài tài liệu. Tiến trình thực tế được theo dõi qua route `GET /status` (poll bảng `ingestion_status`).
{{% /notice %}}

---

#### Nội dung tiếp theo

- [5.4.7 - Route phụ](../5.4.7-Alternative-Route/)
