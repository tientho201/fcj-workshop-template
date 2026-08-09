---
title: "Hạ tầng: EventBridge Scheduler và Lambda Container Image"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.6.1 </b> "
---

#### EventBridge Scheduler gọi thẳng Lambda

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

Khác với Luồng 1 (S3 → SQS → Lambda) và Luồng 2 (API Gateway → Lambda), Luồng 4 dùng **`aws_scheduler_schedule` gọi thẳng Lambda `evaluation_runner`** — không qua SQS hay S3 trung gian nào. Lý do đơn giản: đây là **job chạy theo lịch (batch hàng ngày)**, không phải phản ứng theo sự kiện phát sinh bất kỳ lúc nào như 2 luồng kia, nên không cần lớp buffer/retry qua queue.

#### Lambda đóng gói Container Image (khác 2 Lambda kia)

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
📌 **Vì sao Lambda này đóng gói kiểu `package_type = "Image"` (kéo từ ECR) thay vì `zip` (đóng gói từ `archive_file`) như 2 Lambda kia?** RAGAS kéo theo hàng loạt dependency nặng: `datasets`, `langchain-aws`, `ragas`, `pandas`... — một số trong đó có compiled dependencies (biên dịch native), không phù hợp để "vendor thủ công" (tự tải package vào thư mục rồi zip lại) như cách làm với `pypdf` ở Luồng 1. Đóng gói dưới dạng Docker image giải quyết triệt để vấn đề này, đổi lại phải build/push image trước khi Lambda có thể chạy — xem phần "gate 2 pha" bên dưới.
{{% /notice %}}

#### Gate 2 pha qua `evaluation_image_pushed` — bẫy triển khai quan trọng nhất

{{% notice warning %}}
**Đây là cái bẫy triển khai quan trọng nhất của toàn bộ luồng.** Lambda `package_type = "Image"` **bắt buộc phải có image thật nằm sẵn trong ECR tại thời điểm tạo resource** — nhưng **Terraform không tự build/push Docker image được**. Vì vậy, gần như toàn bộ resource trong `modules/evaluation/main.tf` (Lambda, EventBridge Schedule, IAM Role của scheduler, alarm RAGAS) đều có `count = var.evaluation_image_pushed ? 1 : 0`.
{{% /notice %}}

Quy trình triển khai bắt buộc phải qua **2 lần apply**:

```hcl
variable "evaluation_image_pushed" {
  type    = bool
  default = false
}
```

**Lần apply đầu tiên** (`evaluation_image_pushed = false`): Terraform chỉ tạo được `aws_ecr_repository` (repo rỗng) và IAM Role của Lambda — **chưa có gì thực sự chạy**, vì Lambda/Schedule/Alarm đều có `count = 0`.

```bash
# Sau lần apply đầu tiên, build và push image thủ công:
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

docker build -t rag-dev-evaluation-runner .
docker tag rag-dev-evaluation-runner:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/rag-dev-evaluation-runner:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/rag-dev-evaluation-runner:latest
```

**Sau đó** bật biến `evaluation_image_pushed = true` trong file `.tfvars`, rồi `terraform apply` **lần thứ hai** — lúc này Lambda, Schedule, IAM Role scheduler, và Alarm RAGAS mới thực sự được tạo, vì image đã tồn tại sẵn trong ECR.

---

#### Nội dung tiếp theo:

- [5.6.2 - IAM Permissions và Alarm chất lượng RAG](../5.6.2-IAM-Alarm-RAGAS/)
