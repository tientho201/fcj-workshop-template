---
title: "Resume OCR Mechanism and Error Handling"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.3.5 </b> "
---

This page presents 2 special mechanisms of the `document_processor` Lambda function: a secondary entrypoint for resuming/canceling documents awaiting OCR confirmation, and handling unexpected runtime errors.

#### Secondary Entrypoint: Resume / Cancel OCR

This Lambda function features **2 entrypoints**:

- **Primary entrypoint via SQS** (`event["Records"]`): Asynchronously processes new files uploaded to S3.
- **Secondary entrypoint bypassing SQS** (invoked directly when `event` contains key `"action"`): Receives direct commands from `chat_engine` (endpoint `/documents-decision`) to resume or cancel scanned PDF documents paused in the `awaiting_ocr_confirmation` state (see page [5.3.3 - Text Extraction by File Type](../5.3.3-Text-Extraction/)).

```python
def lambda_handler(event, context):
    # Secondary entrypoint: Direct invoke from chat-engine (always carries key "action")
    if "action" in event:
        return _handle_direct_invoke(event)

    batch_item_failures = []

    # Primary entrypoint: Process record list from SQS
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

The `_handle_direct_invoke` function routes processing based on `action`:

```python
def _handle_direct_invoke(event):
    action = event["action"]
    bucket = event["bucket"]
    key = event["key"]
    document_id = f"{bucket}/{key}"

    if action == "resume_ocr":
        try:
            # force_ocr=True: Skip pypdf reading, invoke Textract OCR directly
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

The `chat-engine` side asynchronously invokes the `document_processor` Lambda when receiving a request from the user at the `/documents-decision` endpoint:

```python
# Extracted from modules/query/lambda_src/chat_engine/handler.py
lambda_client.invoke(
    FunctionName=DOCUMENT_PROCESSOR_FUNCTION_NAME,
    InvocationType="Event", # Asynchronous invocation (fire-and-forget)
    Payload=json.dumps({"action": action, "bucket": bucket, "key": key}).encode("utf-8"),
)
```

This mechanism allows users, via the chat UI (endpoint `/documents-decision`), to actively decide whether to accept the OCR cost for a scanned PDF file, instead of the system automatically OCR-ing every case.

#### Handling Unexpected Runtime Errors (Function DLQ)

Unlike `ingestion_dlq` (automatically populated by SQS after 3 retries — see page [5.3.1 - Infrastructure: S3 and SQS](../5.3.1-Infrastructure-S3-SQS/)), the `document_processor_fn_dlq` queue is actively logged by application code when encountering unexpected runtime errors (code bugs, not corrupt files):

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

Separating these 2 DLQ tiers enables operations teams to quickly distinguish issue types: if a message lands in `ingestion_dlq` → inspect the source document; if it lands in `document_processor_fn_dlq` → examine the detailed traceback to fix source code.

---

#### Next content

Next: [5.3.6 - End-to-End Testing](../5.3.6-End-To-End-Testing/)
