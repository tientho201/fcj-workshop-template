---
title: "Text Extraction by File Type"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3.3 </b> "
---

For each file pushed into SQS, the `_process_s3_object` function in `handler.py` runs sequentially, **writing status updates to the `ingestion_status` table after each step** so that the UI can poll via the `/status` endpoint. The first step in this pipeline is text extraction — branching according to the file extension.

#### Extension Declarations and Text Extraction Functions

```python
OCR_EXTENSIONS = {".png", ".jpg", ".jpeg", ".tiff", ".tif"}
PDF_EXTENSION = ".pdf"
PLAIN_TEXT_EXTENSIONS = {".txt", ".md", ".csv", ".json", ".log", ".htm", ".html"}

def _extension(key):
    _, _, tail = key.rpartition(".")
    return f".{tail.lower()}" if tail and tail != key else ""
```

```python
def _extract_pdf_text(bucket, key):
    body = s3.get_object(Bucket=bucket, Key=key)["Body"].read()
    reader = pypdf.PdfReader(io.BytesIO(body))
    pages = [page.extract_text() or "" for page in reader.pages]
    return "\n".join(pages).strip()
```

```python
def _extract_with_textract(bucket, key):
    response = textract.detect_document_text(
        Document={"S3Object": {"Bucket": bucket, "Name": key}}
    )
    lines = [
        block["Text"]
        for block in response.get("Blocks", [])
        if block.get("BlockType") == "LINE"
    ]
    return "\n".join(lines)
```

```python
def _extract_text(bucket, key):
    extension = _extension(key)

    if extension in OCR_EXTENSIONS:
        logger.info("Extracting text from %s via Textract", key)
        return _extract_with_textract(bucket, key)

    if extension in PLAIN_TEXT_EXTENSIONS or extension == "":
        body = s3.get_object(Bucket=bucket, Key=key)["Body"].read()
        return body.decode("utf-8", errors="replace")

    raise ValueError(
        f"Unsupported file type '{extension}' for key '{key}'..."
    )
```

#### Branching Logic by File Format

```python
def _process_s3_object(bucket, key, force_ocr=False):
    """`force_ocr` is only ever True on the resume half of the PDF
    human-in-the-loop flow (see _handle_direct_invoke) — a human already
    confirmed OCR is worth running, so this call skips straight to Textract
    instead of trying pypdf again."""
    document_id = f"{bucket}/{key}"
    tracer = Tracer()

    _write_status(document_id, key, "processing", tracer)

    extension = _extension(key)
    is_pdf = extension == PDF_EXTENSION

    if is_pdf and not force_ocr:
        text = _extract_pdf_text(bucket, key)
        if not text:
            # Could be a genuinely empty PDF or a scanned one — pypdf can't
            # tell the difference, and OCR costs money per page + only reads
            # a single page via the sync Textract API, so this is a human's
            # call, not the pipeline's. Pause here; chat-engine's
            # /documents-decision route resumes (force_ocr=True) or cancels
            # this same document_id based on what the user picks.
            logger.info("PDF %s has no embedded text layer; awaiting human OCR decision", key)
            _write_status(
                document_id, key, "awaiting_ocr_confirmation", tracer,
                bucket=bucket, object_key=key,
            )
            return
        extraction_detail = f"Đọc trực tiếp lớp text PDF (pypdf) — {len(text):,} ký tự"
    elif is_pdf:  # force_ocr
        text = _extract_with_textract(bucket, key)
        extraction_detail = f"Textract OCR (.pdf, theo xác nhận của người dùng) — {len(text):,} ký tự"
    else:
        text = _extract_text(bucket, key)
        used_ocr = extension in OCR_EXTENSIONS
        extraction_detail = (
            f"Textract OCR ({extension}) — {len(text):,} ký tự"
            if used_ocr
            else f"Đọc trực tiếp ({extension or 'plain'}) — {len(text):,} ký tự"
        )

    tracer.step("extract", "Trích xuất văn bản", detail=extraction_detail)

    if not text.strip():
        logger.warning("No text extracted from %s — nothing to index", key)
        _write_status(document_id, key, "empty", tracer, parent_count=0, child_count=0)
        return
```

The three processing branches operate as follows:

- **PDF** — attempts to extract the embedded text layer first using `pypdf`. If no text is found, it updates the status to `awaiting_ocr_confirmation` and pauses to wait for user confirmation for OCR.
- **Images** (`.png/.jpg/.jpeg/.tiff/.tif`) — uses Textract directly for OCR, as image formats require optical character recognition.
- **Plain Text** (`.txt/.md/.csv/.json/.log/.htm/.html`) — reads content directly from the S3 object.

{{% notice info %}}
**Important logic in the PDF branch:** `pypdf` is tried first because it is free and has no page limit — Textract is invoked only when the result is empty (indicating a scanned document). This provides significant cost optimization compared to blindly performing OCR on all PDF files.
{{% /notice %}}

#### Writing Status to `ingestion_status`

```python
def _write_status(document_id, key, status, tracer, **extra):
    try:
        item = {
            "document_id": document_id,
            "filename": key,
            "status": status,
            "steps": tracer.steps,
            "total_ms": tracer.total_ms,
            "updated_at": int(time.time()),
            "expires_at": int(time.time()) + STATUS_TTL_DAYS * 86400,
        }
        item.update(extra)
        status_table.put_item(Item=item)
    except Exception:
        logger.warning("Could not write ingestion status for %s", document_id, exc_info=True)
```

Each processing step writes its status to the `ingestion_status` table for the UI to poll via the `/status` endpoint. When a scanned PDF is awaiting OCR confirmation, Lambda intentionally does not raise an exception so SQS does not unnecessarily retry the message, updating the status to `awaiting_ocr_confirmation` instead.

#### Amazon Textract for Scanned Documents/Images

```python
import boto3

textract = boto3.client("textract")

def _write_status(document_id, key, status, tracer, **extra):
    """Publish ingestion progress for the UI to poll. Best-effort: this is
    observability, and failing to report progress must never fail ingestion.
    """
    try:
        item = {
            "document_id": document_id,
            "filename": key,
            "status": status,
            "steps": tracer.steps,
            "total_ms": tracer.total_ms,
            "updated_at": int(time.time()),
            "expires_at": int(time.time()) + STATUS_TTL_DAYS * 86400,
        }
        item.update(extra)
        status_table.put_item(Item=item)
    except Exception:  # noqa: BLE001
        logger.warning("Could not write ingestion status for %s", document_id, exc_info=True)
```

#### Manual OCR Confirmation Workflow for Scanned PDFs

When `pypdf` returns an empty string, Lambda **intentionally refrains from raising an exception** so that SQS does not treat this as a failure or perform unnecessary retries — instead, it records the `awaiting_ocr_confirmation` status and halts processing. The user confirms whether to perform OCR on the document via the `/documents-decision` endpoint before Lambda is invoked again to resume (details on this resume mechanism can be found on page [5.3.5 - Resume OCR Mechanism and Error Handling](../5.3.5-Resume-OCR-Error-Handling/)).

---

#### Next content

Next: [5.3.4 - Parent-Child Chunking and Embedding](../5.3.4-Chunking-Embedding/)
