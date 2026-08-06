---
title: "Cache, Guardrails và IAM"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.4.2 </b> "
---

#### ElastiCache Serverless (Redis) — cache exact-match

```hcl
resource "aws_elasticache_serverless_cache" "semantic_cache" {
  name                 = "${local.name_prefix}-semantic-cache"
  engine               = "redis"
  major_engine_version = "7"
  description          = "Semantic cache for chat-engine: exact-match question -> answer, TTL-based."

  user_group_id      = aws_elasticache_user_group.semantic_cache.id
  security_group_ids = [var.elasticache_security_group_id]
  subnet_ids         = var.private_subnet_ids

  cache_usage_limits {
    data_storage {
      maximum = var.elasticache_max_storage_gb
      unit    = "GB"
    }
    ecpu_per_second {
      maximum = var.elasticache_max_ecpu_per_second
    }
  }

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-semantic-cache"
  })
}
```

{{% notice warning %}}
**Xác thực bằng IAM (mint token ngắn hạn), không dùng mật khẩu tĩnh.** Cần cấp `elasticache:Connect` trên **cả 2 phía**: resource cache (`aws_elasticache_serverless_cache`) **và** IAM user/role của Lambda — thiếu 1 trong 2 sẽ báo lỗi `WRONGPASS`, rất dễ nhầm tưởng là lỗi credential/mật khẩu trong khi thực chất là thiếu quyền IAM.
{{% /notice %}}

```hcl
resource "aws_bedrock_guardrail" "chat" {
  name                      = "${local.name_prefix}-chat-guardrail"
  description               = "Guardrail for RAG chat engine content filtering and PII masking"
  blocked_input_messaging   = "Xin lỗi, câu hỏi của bạn vi phạm chính sách nội dung và không thể được xử lý."
  blocked_outputs_messaging = "Xin lỗi, tôi không thể cung cấp câu trả lời cho yêu cầu này."

  content_policy_config {
    filters_config {
      type            = "HATE"
      input_strength  = "MEDIUM"
      output_strength = "MEDIUM"
    }

    filters_config {
      type            = "VIOLENCE"
      input_strength  = "MEDIUM"
      output_strength = "MEDIUM"
    }
  }

  sensitive_information_policy_config {
    pii_entities_config {
      type   = "EMAIL"
      action = "ANONYMIZE"
    }
    pii_entities_config {
      type   = "PHONE"
      action = "ANONYMIZE"
    }
    pii_entities_config {
      type   = "US_SOCIAL_SECURITY_NUMBER"
      action = "BLOCK"
    }
    pii_entities_config {
      type   = "CREDIT_DEBIT_CARD_NUMBER"
      action = "BLOCK"
    }
  }

  dynamic "topic_policy_config" {
    for_each = length(var.guardrail_denied_topics) > 0 ? [1] : []
    content {
      dynamic "topics_config" {
        for_each = var.guardrail_denied_topics
        content {
          name       = topics_config.value.name
          definition = topics_config.value.definition
          examples   = topics_config.value.examples
          type       = "DENY"
        }
      }
    }
  }

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-chat-guardrail"
  })
}
```

Guardrail hoạt động với các cơ chế kiểm duyệt chính:

- **Thông báo phản hồi**: Thiết lập sẵn câu trả lời thân thiện khi người dùng vi phạm chính sách `(blocked_input_messaging)` hoặc khi mô hình đưa ra phản hồi không phù hợp `(blocked_outputs_messaging)`.
- **Nội dung (Content Policy)** — Chặn nội dung thù ghét (HATE) và bạo lực (VIOLENCE) ở mức MEDIUM, áp dụng đồng thời cho cả input (câu hỏi của người dùng) và output (câu trả lời từ mô hình).
- **Thông tin nhạy cảm (PII)** — Tự động che (ANONYMIZE) Email và Số điện thoại; đồng thời chặn cứng (BLOCK) Số An sinh xã hội (US_SOCIAL_SECURITY_NUMBER) và Số thẻ tín dụng (CREDIT_DEBIT_CARD_NUMBER).
- **Chủ đề bị cấm (Topic Policy)** — Cho phép cấu hình danh sách chủ đề cấm tùy chỉnh động qua biến `var.guardrail_denied_topics` (mỗi phần tử chứa name, definition, examples; toàn bộ bị chặn với type = "DENY"). Khối topic_policy_config chỉ được tạo khi danh sách không rỗng — tránh lỗi schema khi triển khai ở môi trường không cần topic filter.

#### IAM least-privilege cho chat_engine

Trong codebase thực tế `modules/query/main.tf`, các quyền IAM (Least Privilege) của chat_engine không gộp chung vào 1 block duy nhất, mà được tách thành từng `aws_iam_policy_document` nhỏ và gắn riêng bằng `aws_iam_role_policy`.

##### 1. Quét Child Chunks (DynamoDB Scan)

```hcl
data "aws_iam_policy_document" "chat_engine_child_chunks" {
  statement {
    sid       = "ScanChildChunks"
    effect    = "Allow"
    actions   = ["dynamodb:Scan"]
    resources = [var.child_chunks_table_arn]
  }
}
resource "aws_iam_role_policy" "chat_engine_child_chunks" {
  name   = "child-chunks"
  role   = aws_iam_role.chat_engine.id
  policy = data.aws_iam_policy_document.chat_engine_child_chunks.json
}
```

##### 2. Gọi Amazon Bedrock (Model & Guardrail)

```hcl
data "aws_iam_policy_document" "chat_engine_bedrock" {
  statement {
    sid     = "InvokeModels"
    effect  = "Allow"
    actions = ["bedrock:InvokeModel"]
    resources = [
      "arn:${data.aws_partition.current.partition}:bedrock:${var.aws_region}::foundation-model/${var.bedrock_embedding_model_id}",
      "arn:${data.aws_partition.current.partition}:bedrock:${var.aws_region}::foundation-model/${var.bedrock_text_model_id}",
      "arn:${data.aws_partition.current.partition}:bedrock:${var.aws_region}::foundation-model/${var.bedrock_haiku_model_id}",
    ]
  }

  statement {
    sid       = "ApplyGuardrail"
    effect    = "Allow"
    actions   = ["bedrock:ApplyGuardrail"]
    resources = [aws_bedrock_guardrail.chat.guardrail_arn]
  }
}
```

##### 3. Đọc/Ghi Lịch sử Chat & Feedback

```hcl
data "aws_iam_policy_document" "chat_engine_dynamodb" {
  statement {
    sid    = "ChatHistoryReadWrite"
    effect = "Allow"
    actions = [
      "dynamodb:PutItem",
      "dynamodb:GetItem",
      "dynamodb:Query",
      "dynamodb:UpdateItem",
    ]
    resources = [aws_dynamodb_table.chat_history.arn]
  }

  statement {
    sid    = "FeedbackReadWrite"
    effect = "Allow"
    actions = [
      "dynamodb:PutItem",
      "dynamodb:GetItem",
      "dynamodb:UpdateItem",
    ]
    resources = [aws_dynamodb_table.feedback.arn]
  }
}
```

##### 4. Đọc Parent Chunks

```hcl
data "aws_iam_policy_document" "chat_engine_parent_chunks" {
  statement {
    sid       = "ReadParentChunks"
    effect    = "Allow"
    actions   = ["dynamodb:GetItem", "dynamodb:BatchGetItem"]
    resources = [var.parent_chunks_table_arn]
  }
}
```

##### 5. Gọi Lambda document_processor

```hcl
data "aws_iam_policy_document" "chat_engine_invoke_document_processor" {
  statement {
    sid       = "InvokeDocumentProcessorForOcrDecision"
    effect    = "Allow"
    actions   = ["lambda:InvokeFunction"]
    resources = [var.document_processor_function_arn]
  }
}
```

##### 6. Kết nối ElastiCache Redis

```hcl
data "aws_iam_policy_document" "chat_engine_elasticache" {
  statement {
    sid     = "ConnectToSemanticCache"
    effect  = "Allow"
    actions = ["elasticache:Connect"]
    resources = [
      aws_elasticache_serverless_cache.semantic_cache.arn,
      aws_elasticache_user.chat_engine.arn,
    ]
  }
}
```

7 điểm đáng chú ý trong IAM Role của `chat_engine`:

| Quyền                                    | Phạm vi (Resource Scope)                         | Mục đích / Lý do                                                                                                                                                 |
| ---------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dynamodb:Scan`                          | Chỉ trên bảng `child_chunks`                     | Giới hạn duy nhất ở bảng chứa vector con để Lambda thực hiện tìm kiếm tương đồng (cosine similarity). Giúp ngăn chặn việc Scan nhầm gây tốn kém ở các bảng khác. |
| `dynamodb:GetItem` / `BatchGetItem`      | Chỉ trên bảng `parent_chunks`                    | Chỉ lấy đúng nội dung văn bản gốc của chunk cha dựa theo `parent_id` sau khi đã tìm thấy các hit từ `child_chunks`.                                              |
| `dynamodb:*` (Read/Write)                | Bảng `chat_history` & `feedback`                 | Cho phép đọc/ghi lịch sử trò chuyện theo `session_id` và lưu đánh giá/phản hồi từ người dùng.                                                                    |
| `bedrock:InvokeModel` & `ApplyGuardrail` | Scoped đúng 3 Model ID & 1 Guardrail ARN         | Giới hạn chính xác trong 3 model (Embedding, Claude sinh câu trả lời, Haiku rewrite query) và 1 Guardrail ARN cụ thể — tuyệt đối không dùng `bedrock:*`.         |
| `s3:PutObject` & `dynamodb:GetItem`      | Bucket `raw_documents` & Bảng `ingestion_status` | Phục vụ 2 endpoint UI hỗ trợ: `POST /documents` (tải tài liệu trực tiếp lên S3) và `GET /status` (poll trạng thái nạp tài liệu).                                 |
| `lambda:InvokeFunction`                  | Scoped đúng 1 ARN của `document_processor`       | Phục vụ endpoint `POST /documents-decision` gọi trực tiếp Lambda xử lý tài liệu (bỏ qua S3/SQS) để tiếp tục hoặc hủy OCR.                                        |
| `elasticache:Connect`                    | Serverless Cache ARN & ElastiCache User ARN      | Bắt buộc khai báo cả 2 resource để Lambda tạo token IAM ngắn hạn đăng nhập Redis an toàn thay vì lưu cứng mật khẩu.                                              |

---

#### Nội dung tiếp theo

- [5.4.3 - Cache lookup và Query Rewriting](../5.4.3-Cache-Lookup-Query-Rewriting/)
