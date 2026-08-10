---
title: "Hạ tầng: S3 và SQS"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.3.1 </b> "
---

Hạ tầng của Luồng 1 được khai báo tại `modules/ingestion/main.tf`. Trang này trình bày 2 thành phần đầu tiên: S3 bucket lưu tài liệu gốc, và SQS đóng vai trò buffer trung gian với 2 tầng Dead Letter Queue.

#### S3 Bucket và vòng đời lưu trữ

Bucket `raw_documents` bật versioning, mã hóa SSE-S3, và có lifecycle rule tự động chuyển object sang Glacier sau 90 ngày để tiết kiệm chi phí lưu trữ dài hạn.

```hcl
resource "aws_s3_bucket" "raw_documents" {
  bucket        = "${local.name_prefix}-raw-documents-${data.aws_caller_identity.current.account_id}"
  force_destroy = var.raw_documents_bucket_force_destroy

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-raw-documents"
  })
}

resource "aws_s3_bucket_versioning" "raw_documents" {
  bucket = aws_s3_bucket.raw_documents.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "raw_documents" {
  bucket = aws_s3_bucket.raw_documents.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "raw_documents" {
  # Versioning must be enabled before a lifecycle configuration referencing
  # noncurrent_version_transition can be applied.
  depends_on = [aws_s3_bucket_versioning.raw_documents]
  bucket = aws_s3_bucket.raw_documents.id
  rule {
    id     = "glacier-after-90-days"
    status = "Enabled"
    filter {}
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }
  }
}
```

#### SQS buffer + 2 tầng Dead Letter Queue

S3 phát event `s3:ObjectCreated:*` **thẳng vào SQS `ingestion_queue`** — không qua Lambda trung gian nào để nhận event, giảm 1 lớp thành phần không cần thiết.

```hcl
resource "aws_sqs_queue" "ingestion_dlq" {
  name                      = "${local.name_prefix}-ingestion-dlq"
  message_retention_seconds = 1209600 # 14 days — max, gives time to investigate/replay
  tags = merge(var.tags, {
    Name = "${local.name_prefix}-ingestion-dlq"
  })
}

resource "aws_sqs_queue" "ingestion_queue" {
  name                       = "${local.name_prefix}-ingestion-queue"
  visibility_timeout_seconds = var.document_processor_timeout_seconds + 30
  message_retention_seconds  = 345600 # 4 days

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.ingestion_dlq.arn
    maxReceiveCount     = 3
  })
  tags = merge(var.tags, {
    Name = "${local.name_prefix}-ingestion-queue"
  })
}

resource "aws_sqs_queue" "document_processor_fn_dlq" {
  name                      = "${local.name_prefix}-document-processor-fn-dlq"
  message_retention_seconds = 1209600

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-document-processor-fn-dlq"
  })
}
```

{{% notice tip %}}
**Có 2 tầng DLQ khác nhau, khác mục đích:**

- **`ingestion_dlq`** — SQS tự động đẩy vào sau 3 lần retry, không cần code app can thiệp. Nghĩa là "file này bị thử lại 3 lần đều fail" (thường do file hỏng/sai định dạng).
- **`document_processor_fn_dlq`** — do chính code Lambda **chủ động ghi** (hàm `_report_to_function_dlq` ở `handler.py:339`) kèm traceback đầy đủ, khi gặp lỗi runtime bất ngờ (không phải lỗi "file hỏng" mà là "code bị bug"). Chi tiết cách dùng ở trang [5.3.5 - Cơ chế Resume OCR và xử lý lỗi](../5.3.5-Resume-OCR-Error-Handling/).
  {{% /notice %}}

`visibility_timeout_seconds` được tính bằng `document_processor_timeout + 30` — đủ để Lambda xử lý xong 1 message trước khi SQS phát lại nó cho consumer khác. `redrive_policy` với `maxReceiveCount = 3` nghĩa là quá 3 lần fail thì message rơi sang `ingestion_dlq`.

#### Kết nối S3 → SQS và Event Source Mapping

```hcl
resource "aws_sqs_queue_policy" "ingestion_queue" {
  queue_url = aws_sqs_queue.ingestion_queue.id
  policy    = data.aws_iam_policy_document.ingestion_queue_policy.json
}

resource "aws_s3_bucket_notification" "raw_documents" {
  bucket = aws_s3_bucket.raw_documents.id

  queue {
    queue_arn = aws_sqs_queue.ingestion_queue.arn
    events    = ["s3:ObjectCreated:*"]
  }

  depends_on = [aws_sqs_queue_policy.ingestion_queue]
}

resource "aws_lambda_event_source_mapping" "document_processor" {
  event_source_arn        = aws_sqs_queue.ingestion_queue.arn
  function_name           = aws_lambda_function.document_processor.arn
  batch_size              = 1
  function_response_types = ["ReportBatchItemFailures"]
}
```

`event_source_mapping` khai báo `batch_size = 1` — Lambda xử lý từng file một, không gom batch, để một file lỗi không kéo file khác theo.

Test thử bằng cách upload 1 file bất kỳ lên bucket, kiểm tra message đã xuất hiện trong SQS (mục **Send and receive messages → Poll for messages**).

---

#### Nội dung tiếp theo

Tiếp theo: [5.3.2 - Hạ tầng: DynamoDB và IAM Permissions](../5.3.2-Infrastructure-DynamoDB-IAM/)
