---
title: "Infrastructure: DynamoDB and IAM Permissions"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.3.2 </b> "
---

{{% notice note %}}
📌 Compared to the initial architecture diagram, this pipeline **does not use Amazon OpenSearch Serverless**. All vectors and BM25 indices are stored directly in **Amazon DynamoDB**, and Pipeline 2 calculates Hybrid Search (cosine similarity + BM25 → Reciprocal Rank Fusion) at the application layer instead of relying on a dedicated search engine.
{{% /notice %}}

#### 3 DynamoDB Tables Replacing OpenSearch

Instead of "Store Vectors → OpenSearch" as in the original diagram, the project uses 3 DynamoDB tables:

| Table              | Stored Data                                         | Purpose                                                              |
| ------------------ | --------------------------------------------------- | -------------------------------------------------------------------- |
| `ingestion_status` | Processing status per `document_id` after each step | Enables the UI to poll via the `/status` endpoint                    |
| `parent_chunks`    | Raw text of parent chunks (~1000-1500 tokens)       | Broad context lookup using `parent_id` during Q&A                    |
| `child_chunks`     | Binary-packed vectors + BM25 term frequencies       | Data for Pipeline 2 to calculate Hybrid Search (cosine + BM25 → RRF) |

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

#### Least-Privilege IAM Role for document_processor

The Lambda role has strictly necessary permissions — avoid broad permissions like the policy deployed in the environment setup phase (section 5.2):

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
    resources = [var.bedrock_embedding_model_arn] # scoped to exactly 1 model
  }
  statement {
    sid       = "TextractOcr"
    actions   = ["textract:AnalyzeDocument", "textract:DetectDocumentText"]
    resources = ["*"] # exception: these 2 APIs do not support ARN scoping
  }
}

resource "aws_iam_role_policy" "document_processor" {
  name   = "document-processor-permissions"
  role   = aws_iam_role.document_processor.id
  policy = data.aws_iam_policy_document.document_processor_permissions.json
}
```

{{% notice tip %}}
`Textract` is the only exception using `resources = "*"` in this role — because `AnalyzeDocument` and `DetectDocumentText` do not support scoping by ARN. This should be explicitly noted in the code/report so reviewers do not mistake it for a security oversight.
{{% /notice %}}

adRawDocuments`, `WriteScopedDynamoTables`, `InvokeEmbeddingModelOnly`, `TextractOcr`).\_

---

#### Next content

Next: [5.3.3 - Text Extraction by File Type](../5.3.3-Text-Extraction/)
