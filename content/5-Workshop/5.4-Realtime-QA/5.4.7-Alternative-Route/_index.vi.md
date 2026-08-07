---
title: "Route phụ"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 5.4.7 </b> "
---

#### Route theo dõi tiến trình

```python
def _json_safe(value):
    if isinstance(value, decimal.Decimal):
        return int(value) if value == value.to_integral_value() else float(value)
    raise TypeError(f"Not JSON serialisable: {type(value)}")

def _handle_status(event):
    """GET /status?document_id=... — report ingestion progress for the UI.

    Ingestion is asynchronous, so the browser polls this until status leaves
    'processing'. Returns 'pending' (rather than 404) while the S3 event is
    still in flight and document-processor hasn't written its first row yet.
    """
    params = event.get("queryStringParameters") or {}
    document_id = params.get("document_id")
    if not document_id:
        return _response(400, {"error": "Cần có tham số 'document_id'."})

    item = ingestion_status_table.get_item(Key={"document_id": document_id}).get("Item")
    if not item:
        return _response(200, {"status": "pending", "steps": [], "document_id": document_id})

    # DynamoDB hands numbers back as Decimal, which json.dumps refuses.
    return _response(200, json.loads(json.dumps(item, default=_json_safe)))

```

#### Route /upload

```python
def _handle_upload(event):
    """POST /documents — accept a document from the browser and drop it in S3.

    The browser posts the file through this endpoint rather than PUTting
    straight to a presigned S3 URL. That trades a little payload overhead
    (API Gateway caps bodies at 10MB, and base64 inflates that by ~33%) for
    two things worth having: no CORS configuration on the documents bucket,
    and no presigned-URL endpoint whose signature would carry this role's S3
    write permission out to the client.

    Two payload shapes are accepted:
      - `content_base64` (+ optional `content_type`): raw bytes, base64-
        encoded. Required for anything the browser can't safely represent as
        a JSON text string — PDFs, images — so document-processor's existing
        Textract OCR branch (see OCR_EXTENSIONS there) receives the real
        bytes instead of bytes already mangled by a UTF-8 decode attempt.
      - `content`: plain text. Kept for the plain-text file types (.txt,
        .md, ...) and pasted-in content, where JSON-as-text is lossless.

    Writing the object is all this does — the S3 notification then drives the
    existing SQS -> document-processor pipeline unchanged; that pipeline
    picks its read path purely from the key's extension, so nothing
    downstream needs to know which of the two shapes above was used.
    """
    body = json.loads(event.get("body") or "{}")
    filename = (body.get("filename") or "").strip()

    if not filename:
        return _response(400, {"error": "Cần có 'filename'."})

    content_base64 = body.get("content_base64")
    if content_base64:
        try:
            data = base64.b64decode(content_base64, validate=True)
        except (binascii.Error, ValueError):
            return _response(400, {"error": "'content_base64' không phải base64 hợp lệ."})
        if not data:
            return _response(400, {"error": "Tệp rỗng."})
    else:
        content = body.get("content") or ""
        if not content.strip():
            return _response(400, {"error": "Cần có 'content' hoặc 'content_base64'."})
        data = content.encode("utf-8")

    # Keep the key flat and predictable; strip any path the browser sent.
    safe_name = os.path.basename(filename).replace("\\", "_")
    extension = os.path.splitext(safe_name)[1].lower()
    content_type = (
        body.get("content_type")
        or _CONTENT_TYPE_BY_EXTENSION.get(extension)
        or "application/octet-stream"
    )

    s3.put_object(
        Bucket=RAW_DOCUMENTS_BUCKET,
        Key=safe_name,
        Body=data,
        ContentType=content_type,
    )

    return _response(
        202,
        {
            "document_id": f"{RAW_DOCUMENTS_BUCKET}/{safe_name}",
            "filename": safe_name,
            "bytes": len(data),
            "message": "Đã tải lên S3. Pipeline ingestion đang chạy bất đồng bộ.",
        },
    )
```

{{% notice note %}}
Hàm \_handle_upload(event) nằm ở `handler.py:272–340`
, chịu trách nhiệm xử lý endpoint POST /documents để giao diện người dùng (Browser UI) tải file tài liệu trực tiếp lên S3 bucket.
{{% /notice %}}

1. Đánh đổi thiết kế (Design Trade-offs)
   Thay vì cấp một Presigned S3 URL cho trình duyệt tải thẳng file lên S3, hệ thống đẩy dữ liệu qua API Gateway + Lambda:
   - Đánh đổi: Tốn thêm một chút overhead truyền tải (API Gateway giới hạn body tối đa 10MB và mã hóa base64 làm phình dung lượng ~33%).
   - Lợi ích:
     - Không cần bật CORS rộng rãi trên S3 bucket raw_documents.
     - Bảo mật hơn: Tránh việc tạo endpoint cấp Presigned-URL có thể bị rò rỉ quyền ghi S3 ra phía Client.

2. Hai định dạng Payload chấp nhận
   - content_base64 (Ưu tiên cho file nhị phân: PDF, PNG, JPG, TIFF):
     - Mã hóa base64 chuỗi bytes gốc của file.
     - Đảm bảo các file PDF/Ảnh không bị lỗi do trình duyệt cố giải mã UTF-8, giữ nguyên vẹn byte stream để chuyển cho AWS Textract OCR xử lý ở Luồng 1.
   - content (Dành cho file văn bản: TXT, MD, CSV, JSON):
     - Chứa nội dung văn bản thuần hoặc văn bản copy-paste trực tiếp từ giao diện.

#### Route: /feedback

```python
def _handle_feedback(event):
    """POST /feedback — thumbs up/down on a single chat answer.

    `message_id` is the id /chat generated and returned with that specific
    answer (see _handle_chat) — not a reference into chat_history, which has
    no lookup path by message_id today (see _persist_turn's comment). This
    table was provisioned with IAM access from day one but had no caller
    until this route; modules/evaluation's evaluation-runner is the reader.
    """
    body = json.loads(event.get("body") or "{}")
    message_id = (body.get("message_id") or "").strip()
    rating = body.get("rating")
    comment = (body.get("comment") or "").strip()

    if not message_id:
        return _response(400, {"error": "Cần có 'message_id'."})
    if rating not in ("up", "down"):
        return _response(400, {"error": "'rating' phải là 'up' hoặc 'down'."})

    now = datetime.datetime.now(datetime.timezone.utc)
    feedback_table.put_item(
        Item={
            "message_id": message_id,
            "rating": rating,
            "comment": comment,
            # Date-only prefix so modules/evaluation's daily batch job can
            # filter "yesterday's records" with begins_with(created_at, ...).
            "created_at": now.date().isoformat(),
            "created_at_iso": now.isoformat(),
        }
    )

    return _response(202, {"message_id": message_id, "rating": rating, "message": "Đã ghi nhận phản hồi."})

```

---

#### Nội dung tiếp theo

- [5.4.8 - Kiểm thử end-to-end](../5.4.8-End-To-End-Testing/)
