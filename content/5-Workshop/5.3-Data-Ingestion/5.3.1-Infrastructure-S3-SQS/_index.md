---
title: "Infrastructure: S3 and SQS"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.3.1 </b> "
---

Pipeline 1's infrastructure is declared in `modules/ingestion/main.tf`. This page presents the first 2 components: the S3 bucket for storing original documents, and SQS acting as an intermediate buffer with 2 levels of Dead Letter Queues.

#### S3 Bucket and Lifecycle Management

The `raw_documents` bucket enables versioning, SSE-S3 encryption, and includes a lifecycle rule to automatically transition objects to Glacier after 90 days to optimize long-term storage costs.

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

#### SQS Buffer + 2-Tier Dead Letter Queue

S3 sends `s3:ObjectCreated:*` events **directly into SQS `ingestion_queue`** — without an intermediate Lambda function to receive events, eliminating an unnecessary architectural layer.

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
**There are 2 distinct DLQ levels with different purposes:**
- **`ingestion_dlq`** — Automatically populated by SQS after 3 failed retries without application code intervention. This means "this file failed after 3 retries" (typically due to corrupt or invalid file format).
- **`document_processor_fn_dlq`** — **Actively logged** by the Lambda code itself (`_report_to_function_dlq` function in `handler.py:339`) with full traceback when encountering unexpected runtime errors (not a "corrupt file" error, but a code bug). Details on usage can be found on page [5.3.5 - Resume OCR Mechanism and Error Handling](../5.3.5-Resume-OCR-Error-Handling/).
{{% /notice %}}

`visibility_timeout_seconds` is calculated as `document_processor_timeout + 30` — sufficient for Lambda to finish processing 1 message before SQS re-delivers it to another consumer. `redrive_policy` with `maxReceiveCount = 3` means after failing 3 times, the message is sent to `ingestion_dlq`.

#### S3 → SQS Connection and Event Source Mapping

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

`event_source_mapping` specifies `batch_size = 1` — Lambda processes files individually rather than batching, preventing a failing file from impacting others.

Test by uploading any file to the bucket and verifying that the message appears in SQS (under **Send and receive messages → Poll for messages**).

---
#### Next content
- [5.3.2 - Infrastructure: DynamoDB and IAM Permissions](../5.3.2-Infrastructure-DynamoDB-IAM/)
