---
title: "Error Handling and OCR Decision"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.4.6 </b> "
---

In addition to the main `_handle_chat` route presented in the previous 3 pages, the `chat_engine` Lambda also handles specific errors from Bedrock and includes auxiliary routes connecting back to Stream 1.

#### Dedicated Handling for Bedrock Throttling Errors

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
Rather than letting Bedrock Throttling errors mingle into generic 500 internal server errors, the code **specifically catches** `ThrottlingException`/`TooManyRequestsException` and returns **HTTP 429** with `{"retryable": true}` — helping the client (frontend) know precisely that it should automatically retry after a brief delay, instead of displaying a generic system error to users. Simultaneously, it logs at **ERROR** level so that **Stream 3's CloudWatch Metric Filter** catches it and triggers the `CW Alarm Bedrock Throttle` alarm (configured in the Monitoring section).
{{% /notice %}}

#### Auxiliary Route: `/documents-decision` — Connecting to Stream 1

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

The `_handle_ocr_decision` function (`handler.py:369-410`) calls `lambda_client.invoke(..., InvocationType="Event")` to **invoke `document_processor` (Stream 1's Lambda) asynchronously** — this serves as the exact connecting point between the two streams for the manual OCR confirmation mechanism described on [page 5.3.5](../../5.3-Data-Ingestion/5.3.5-Resume-OCR-Error-Handling/).

{{% notice note %}}
Using `InvocationType="Event"` (asynchronous) instead of `RequestResponse` (synchronous) is critical: users clicking "Confirm OCR" on the interface only need to receive an immediate response (202 Accepted) without waiting for the entire OCR + embedding + DynamoDB storage process to complete — a process that can take anywhere from a few seconds to tens of seconds depending on document length. Actual progress is tracked via the `GET /status` route (polling the `ingestion_status` table).
{{% /notice %}}

---

#### Next Content

- [5.4.7 - Alternative Route](../5.4.7-Alternative-Route/)
