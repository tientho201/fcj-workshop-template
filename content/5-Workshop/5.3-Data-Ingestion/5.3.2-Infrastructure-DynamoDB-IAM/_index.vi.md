---
title: "Hạ tầng: DynamoDB và IAM Permissions"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
---

{{% notice note %}}
📌 So với sơ đồ kiến trúc ban đầu, luồng này **không dùng Amazon OpenSearch Serverless**. Toàn bộ vector và chỉ số BM25 được lưu trực tiếp trong **Amazon DynamoDB**, và Luồng 2 sẽ tự tính Hybrid Search (cosine similarity + BM25 → Reciprocal Rank Fusion) ở tầng ứng dụng thay vì dựa vào engine tìm kiếm chuyên dụng.
{{% /notice %}}

#### 3 bảng DynamoDB thay thế OpenSearch

Thay vì "Lưu Vectors → OpenSearch" như sơ đồ gốc, dự án dùng 3 bảng DynamoDB:

| Bảng               | Dữ liệu lưu                                      | Mục đích                                                    |
| ------------------ | ------------------------------------------------ | ----------------------------------------------------------- |
| `ingestion_status` | Trạng thái xử lý theo `document_id` sau mỗi bước | Cho UI poll qua endpoint `/status`                          |
| `parent_chunks`    | Text thô của parent chunk (~1000-1500 token)     | Tra cứu ngữ cảnh rộng bằng `parent_id` lúc trả lời câu hỏi  |
| `child_chunks`     | Vector đóng gói nhị phân + BM25 term frequencies | Dữ liệu để Luồng 2 tính Hybrid Search (cosine + BM25 → RRF) |

```hcl
resource "aws_dynamodb_table" "parent_chunks" {
  name         = "${local.name_prefix}-parent-chunks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "parent_id"
  attribute {
    name = "parent_id"
    type = "S"
  }
  point_in_time_recovery {
    enabled = true
  }
  tags = merge(var.tags, {
    Name = "${local.name_prefix}-parent-chunks"
  })
}

resource "aws_dynamodb_table" "child_chunks" {
  name         = "${local.name_prefix}-child-chunks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "chunk_id"
  attribute {
    name = "chunk_id"
    type = "S"
  }
  attribute {
    name = "document_id"
    type = "S"
  }
  global_secondary_index {
    name            = "document-id-index"
    hash_key        = "document_id"
    projection_type = "KEYS_ONLY"
  }
  point_in_time_recovery {
    enabled = true
  }
  tags = merge(var.tags, {
    Name = "${local.name_prefix}-child-chunks"
  })
}

resource "aws_dynamodb_table" "ingestion_status" {
  name         = "${local.name_prefix}-ingestion-status"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "document_id"

  attribute {
    name = "document_id"
    type = "S"
  }

  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-ingestion-status"
  })
}
```

#### IAM Role least-privilege cho document_processor

Role của Lambda chỉ có đúng những quyền cần thiết — không dùng quyền rộng như policy deploy ở phần chuẩn bị môi trường (mục 5.2):

```hcl
data "aws_iam_policy_document" "document_processor_permissions" {
  statement {
    sid       = "ReadRawDocuments"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.raw_documents.arn}/*"]
  }
  statement {
    sid     = "WriteScopedDynamoTables"
    actions = ["dynamodb:PutItem", "dynamodb:GetItem", "dynamodb:Query", "dynamodb:DeleteItem", "dynamodb:UpdateItem"]
    resources = [
      aws_dynamodb_table.parent_chunks.arn,
      aws_dynamodb_table.child_chunks.arn,
      aws_dynamodb_table.ingestion_status.arn,
    ]
  }
  statement {
    sid       = "InvokeEmbeddingModelOnly"
    actions   = ["bedrock:InvokeModel"]
    resources = [var.bedrock_embedding_model_arn] # scoped đúng 1 model
  }
  statement {
    sid       = "TextractOcr"
    actions   = ["textract:AnalyzeDocument", "textract:DetectDocumentText"]
    resources = ["*"] # ngoại lệ: 2 API này không hỗ trợ ARN-scoping
  }
}

resource "aws_iam_role_policy" "document_processor" {
  name   = "document-processor-permissions"
  role   = aws_iam_role.document_processor.id
  policy = data.aws_iam_policy_document.document_processor_permissions.json
}
```

{{% notice tip %}}
`Textract` là ngoại lệ duy nhất dùng `resources = "*"` trong role này — vì `AnalyzeDocument` và `DetectDocumentText` không hỗ trợ scoping theo ARN. Đây là điểm nên ghi chú rõ trong code/báo cáo để người review không hiểu nhầm là thiếu sót về bảo mật.
{{% /notice %}}

---

#### Nội dung tiếp theo

Tiếp theo: [5.3.3 - Trích xuất văn bản theo loại file](../5.3.3-Text-Extraction/)
