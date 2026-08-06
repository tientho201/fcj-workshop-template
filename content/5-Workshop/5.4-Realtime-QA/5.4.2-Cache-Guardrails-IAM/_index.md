---
title: "Cache, Guardrails and IAM"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.4.2 </b> "
---

#### ElastiCache Serverless (Redis) — Exact-Match Cache

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
**IAM authentication (minting short-lived tokens) is used instead of static passwords.** `elasticache:Connect` permissions must be granted on **both sides**: the cache resource (`aws_elasticache_serverless_cache`) **and** the Lambda's IAM user/role — missing either will result in a `WRONGPASS` error, easily mistaken for credential/password errors when it is actually a missing IAM permission.
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

Guardrails operate with key moderation mechanisms:

- **Response Messaging**: Pre-configured user-friendly responses when users violate content policy `(blocked_input_messaging)` or when the model produces inappropriate output `(blocked_outputs_messaging)`.
- **Content Policy** — Blocks HATE and VIOLENCE content at MEDIUM strength, applied simultaneously to both input (user questions) and output (model responses).
- **Sensitive Information (PII)** — Automatically masks (ANONYMIZE) Emails and Phone numbers; hard-blocks (BLOCK) US Social Security Numbers (US_SOCIAL_SECURITY_NUMBER) and Credit/Debit Card Numbers (CREDIT_DEBIT_CARD_NUMBER).
- **Denied Topics (Topic Policy)** — Allows dynamic custom denied topic configuration via `var.guardrail_denied_topics` (each element contains name, definition, examples; all blocked with type = "DENY"). The topic_policy_config block is only created when the list is non-empty — avoiding schema errors when deploying in environments that do not require topic filtering.

#### Least-Privilege IAM for chat_engine

In the actual `modules/query/main.tf` codebase, least-privilege IAM permissions for chat_engine are not bundled into a single policy block, but separated into distinct `aws_iam_policy_document` definitions attached via `aws_iam_role_policy`.

##### 1. Scanning Child Chunks (DynamoDB Scan)

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

##### 2. Invoking Amazon Bedrock (Models & Guardrails)

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

##### 3. Reading/Writing Chat History & Feedback

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

##### 4. Reading Parent Chunks

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

##### 5. Invoking document_processor Lambda

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

##### 6. Connecting to ElastiCache Redis

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

7 key aspects of the `chat_engine` IAM Role:

| Permission                               | Resource Scope                                   | Purpose / Rationale                                                                                                                                              |
| ---------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dynamodb:Scan`                          | `child_chunks` table only                        | Restricted solely to the table containing child vectors so Lambda can perform similarity search (cosine similarity). Prevents accidental costly scans on other tables. |
| `dynamodb:GetItem` / `BatchGetItem`      | `parent_chunks` table only                       | Fetches exact original text content of parent chunks by `parent_id` after hits are identified in `child_chunks`.                                                 |
| `dynamodb:*` (Read/Write)                | `chat_history` & `feedback` tables               | Enables reading/writing conversation history by `session_id` and saving user feedback/ratings.                                                                  |
| `bedrock:InvokeModel` & `ApplyGuardrail` | Scoped precisely to 3 Model IDs & 1 Guardrail ARN| Strictly limited to 3 models (Embedding, Claude answer generator, Haiku query rewriter) and 1 specific Guardrail ARN — never uses wildcard `bedrock:*`.          |
| `s3:PutObject` & `dynamodb:GetItem`      | `raw_documents` bucket & `ingestion_status` table| Powers 2 supporting UI endpoints: `POST /documents` (direct document upload to S3) and `GET /status` (polling document ingestion status).                        |
| `lambda:InvokeFunction`                  | Scoped strictly to 1 `document_processor` ARN    | Powers `POST /documents-decision` endpoint to directly invoke the document processor Lambda (bypassing S3/SQS) to resume or cancel OCR.                       |
| `elasticache:Connect`                    | Serverless Cache ARN & ElastiCache User ARN      | Mandatory to declare both resources so Lambda can generate short-lived IAM tokens to securely log into Redis instead of hardcoding passwords.                  |

---

#### Next Content

- [5.4.3 - Cache Lookup and Query Rewriting](../5.4.3-Cache-Lookup-Query-Rewriting/)
