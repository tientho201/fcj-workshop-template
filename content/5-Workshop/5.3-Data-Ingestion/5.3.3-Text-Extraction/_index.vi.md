---
title: "Trích xuất văn bản theo loại file"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3.3 </b> "
---

Với mỗi file rơi vào SQS, hàm `_process_s3_object` trong `handler.py` chạy tuần tự, **ghi trạng thái vào bảng `ingestion_status` sau mỗi bước** để UI poll qua endpoint `/status`. Bước đầu tiên trong chuỗi này là trích xuất văn bản — rẽ nhánh theo phần mở rộng của file.

#### Khai báo extensions và các hàm trích xuất text

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

#### Logic rẽ nhánh theo định dạng file

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

Ba nhánh xử lý như sau:

- **PDF** — thử đọc lớp text có sẵn bằng `pypdf` trước. Nếu không có text thì ghi trạng thái `awaiting_ocr_confirmation` và dừng lại để chờ người dùng xác nhận OCR.
- **Ảnh** (`.png/.jpg/.jpeg/.tiff/.tif`) — dùng Textract để OCR trực tiếp, vì đây là định dạng cần chuyển từ hình ảnh sang văn bản.
- **Text thường** (`.txt/.md/.csv/.json/.log/.htm/.html`) — đọc trực tiếp nội dung từ object S3.

{{% notice info %}}
**Logic quan trọng ở nhánh PDF:** thử `pypdf` trước vì miễn phí và không giới hạn số trang — chỉ khi kết quả rỗng (dấu hiệu tài liệu scan) mới cần tới Textract. Đây là tối ưu chi phí quan trọng so với việc OCR mù mọi file PDF.
{{% /notice %}}

#### Ghi trạng thái `ingestion_status`

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

Mỗi bước xử lý sẽ ghi trạng thái xuống bảng `ingestion_status` để UI poll qua endpoint `/status`. Khi PDF scan chờ xác nhận OCR, Lambda không raise exception để SQS không retry vô ích mà chỉ cập nhật trạng thái thành `awaiting_ocr_confirmation`.

#### Amazon Textract cho tài liệu scan/ảnh

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

#### Luồng xác nhận OCR thủ công cho PDF scan

Khi `pypdf` trả về chuỗi rỗng, Lambda **cố tình không raise exception** để SQS không coi đây là lỗi và không retry vô ích — thay vào đó ghi status `awaiting_ocr_confirmation` và dừng xử lý. Người dùng sẽ xác nhận có muốn OCR tài liệu này hay không thông qua endpoint `/documents-decision`, trước khi Lambda được gọi lại để tiếp tục (chi tiết cơ chế gọi lại này ở trang [5.3.5 - Cơ chế Resume OCR và xử lý lỗi](../5.3.5-Resume-OCR-Error-Handling/)).

---

#### Nội dung tiếp theo

Tiếp theo: [5.3.4 - Parent-Child Chunking và Embedding](../5.3.4-Chunking-Embedding/)
