---
title: "Infrastructure: EventBridge Scheduler and Lambda Container Image"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.6.1 </b> "
---

#### EventBridge Scheduler directly invokes Lambda

```hcl
resource "aws_scheduler_schedule" "evaluation_daily" {
  count = var.evaluation_image_pushed ? 1 : 0

  name                = "${local.name_prefix}-evaluation-daily"
  schedule_expression = var.evaluation_schedule_expression

  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = aws_lambda_function.evaluation_runner[0].arn
    role_arn = aws_iam_role.scheduler[0].arn
  }
}
```

Unlike Stream 1 (S3 → SQS → Lambda) and Stream 2 (API Gateway → Lambda), Stream 4 uses **`aws_scheduler_schedule` to directly invoke the `evaluation_runner` Lambda** — without any SQS or S3 middleware. The reason is simple: this is a **scheduled job (daily batch)**, not a reaction to events arising at arbitrary times like the other two streams, so buffer/retry layers via queues are unnecessary.

#### Lambda packaged as Container Image (different from the other two Lambdas)

```hcl
resource "aws_lambda_function" "evaluation_runner" {
  count = var.evaluation_image_pushed ? 1 : 0

  function_name = local.evaluation_fn_name
  role          = aws_iam_role.evaluation_runner.arn
  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.evaluation_runner.repository_url}:${var.evaluation_image_tag}"
  memory_size   = var.evaluation_memory_mb
  timeout       = var.evaluation_timeout_seconds

  environment {
    variables = {
      CHAT_HISTORY_TABLE     = var.chat_history_table_name
      FEEDBACK_TABLE         = var.feedback_table_name
      RESULTS_BUCKET         = aws_s3_bucket.evaluation_results.bucket
      RAGAS_JUDGE_MODEL_ID   = var.ragas_judge_model_id
      FAITHFULNESS_THRESHOLD = tostring(var.ragas_faithfulness_threshold)
    }
  }

  tags = merge(var.tags, {
    Name = local.evaluation_fn_name
  })

  depends_on = [aws_cloudwatch_log_group.evaluation_runner]
}

resource "aws_ecr_repository" "evaluation_runner" {
  name                 = "${local.name_prefix}-evaluation-runner"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-evaluation-runner"
  })
}
```

{{% notice note %}}
📌 **Why is this Lambda packaged with `package_type = "Image"` (pulled from ECR) instead of `zip` (packaged via `archive_file`) like the other two Lambdas?** RAGAS pulls in a heavy stack of dependencies: `datasets`, `langchain-aws`, `ragas`, `pandas`... — several of which contain compiled native dependencies, making them unsuitable for "manual vendoring" (manually downloading packages into a directory and zipping them up) like the approach used with `pypdf` in Stream 1. Packaging as a Docker image completely solves this issue, at the cost of requiring the image to be built and pushed before the Lambda can be created — see the "2-phase gate" section below.
{{% /notice %}}

#### 2-Phase Gate via `evaluation_image_pushed` — the most critical deployment trap

{{% notice warning %}}
**This is the single most critical deployment trap of the entire stream.** A Lambda with `package_type = "Image"` **strictly requires a real image to already exist in ECR at resource creation time** — but **Terraform cannot automatically build or push Docker images**. Therefore, nearly all resources in `modules/evaluation/main.tf` (Lambda, EventBridge Schedule, scheduler IAM Role, RAGAS alarm) use `count = var.evaluation_image_pushed ? 1 : 0`.
{{% /notice %}}

The deployment process strictly requires **two `apply` passes**:

```hcl
variable "evaluation_image_pushed" {
  type    = bool
  default = false
}
```

**The first `apply` pass** (`evaluation_image_pushed = false`): Terraform creates only the `aws_ecr_repository` (empty repo) and the Lambda IAM Role — **nothing is actually running yet**, because Lambda/Schedule/Alarm all have `count = 0`.

```bash
# After the first apply pass, manually build and push the image:
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

docker build -t rag-dev-evaluation-runner .
docker tag rag-dev-evaluation-runner:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/rag-dev-evaluation-runner:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/rag-dev-evaluation-runner:latest
```

**Then**, enable `evaluation_image_pushed = true` in your `.tfvars` file, and run `terraform apply` a **second time** — at this point, the Lambda, Schedule, scheduler IAM Role, and RAGAS Alarm are actually created, because the image already exists in ECR.

---

#### Next Content:

- [5.6.2 - IAM Permissions and RAG Quality Alarm](../5.6.2-IAM-Alarm-RAGAS/)
