---
title: "IAM Permissions và Alarm chất lượng RAG"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.6.2 </b> "
---

#### IAM Role cho evaluation_runner

```hcl
data "aws_iam_policy_document" "evaluation_runner_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "evaluation_runner" {
  name               = "${local.name_prefix}-evaluation-runner-role"
  assume_role_policy = data.aws_iam_policy_document.evaluation_runner_assume.json
  tags = var.tags
}

data "aws_iam_policy_document" "evaluation_runner_logs" {
  statement {
    sid       = "WriteOwnLogGroup"
    effect    = "Allow"
    actions   = ["logs:CreateLogStream", "logs:PutLogEvents"]
    resources = ["${aws_cloudwatch_log_group.evaluation_runner.arn}:*"]
  }
}

resource "aws_iam_role_policy" "evaluation_runner_logs" {
  name   = "logs"
  role   = aws_iam_role.evaluation_runner.id
  policy = data.aws_iam_policy_document.evaluation_runner_logs.json
}

data "aws_iam_policy_document" "evaluation_runner_dynamodb" {
  statement {
    sid    = "ReadFeedbackAndChatHistory"
    effect = "Allow"
    actions = [
      "dynamodb:Scan",
      "dynamodb:Query",
      "dynamodb:GetItem",
    ]
    resources = [
      var.feedback_table_arn,
      var.chat_history_table_arn,
    ]
  }
}

resource "aws_iam_role_policy" "evaluation_runner_dynamodb" {
  name   = "dynamodb-read"
  role   = aws_iam_role.evaluation_runner.id
  policy = data.aws_iam_policy_document.evaluation_runner_dynamodb.json
}

data "aws_iam_policy_document" "evaluation_runner_s3" {
  statement {
    sid       = "WriteEvaluationResults"
    effect    = "Allow"
    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.evaluation_results.arn}/*"]
  }
}

resource "aws_iam_role_policy" "evaluation_runner_s3" {
  name   = "s3-write"
  role   = aws_iam_role.evaluation_runner.id
  policy = data.aws_iam_policy_document.evaluation_runner_s3.json
}

data "aws_iam_policy_document" "evaluation_runner_bedrock" {
  statement {
    sid       = "InvokeJudgeModel"
    effect    = "Allow"
    actions   = ["bedrock:InvokeModel"]
    resources = ["arn:${data.aws_partition.current.partition}:bedrock:${var.aws_region}::foundation-model/${var.ragas_judge_model_id}"]
  }
}

resource "aws_iam_role_policy" "evaluation_runner_bedrock" {
  name   = "bedrock"
  role   = aws_iam_role.evaluation_runner.id
  policy = data.aws_iam_policy_document.evaluation_runner_bedrock.json
}

```

6 nhóm quyền:

| Quyền                               | Phạm vi                                                                              | Ghi chú                                                     |
| ----------------------------------- | ------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| `sts:AssumeRole`                    | Service `lambda.amazonaws.com`                                                       | Cho phép dịch vụ AWS Lambda đảm nhận (assume) IAM Role này  |
| `logs:CreateLogStream/PutLogEvents` | Log Group `/aws/lambda/${local.evaluation_fn_name}`                                  | Ghi log thực thi của Lambda ra CloudWatch Logs              |
| `dynamodb:Scan/Query/GetItem`       | `feedback_table` + `chat_history_table` (2 bảng của Luồng 2)                         | Đọc dữ liệu lịch sử chat và phản hồi người dùng để đánh giá |
| `s3:PutObject`                      | Riêng bucket `evaluation_results`                                                    | Ghi file kết quả đánh giá chi tiết (JSON/CSV)               |
| `bedrock:InvokeModel`               | Scoped đúng 1 model (`ragas_judge_model_id`)                                         | Cho phép gọi Bedrock LLM làm "giám khảo" chấm điểm RAGAS    |
| `cloudwatch:PutMetricData`          | `resources = "*"` **nhưng** kèm condition `cloudwatch:namespace = ["RAGEvaluation"]` | Push các chỉ số điểm số RAGAS lên CloudWatch Custom Metrics |
|                                     |

{{% notice tip %}}
**Vì sao `cloudwatch:PutMetricData` phải dùng `resources = "*"`?** Đây là 1 trong số ít action của AWS **không hỗ trợ scope theo ARN** — bản thân action này không có khái niệm "resource" cụ thể để giới hạn. Thay vào đó, dự án dùng **IAM Condition** ràng buộc `cloudwatch:namespace = RAGEvaluation` — nghĩa là Lambda này **chỉ được phép ghi metric vào đúng 1 namespace `RAGEvaluation`**, không thể ghi vào namespace khác (kể cả namespace hệ thống của AWS). Đây vẫn là least-privilege đúng nghĩa, chỉ là cơ chế giới hạn khác với các action khác (dùng `Condition` thay vì `Resource`).
{{% /notice %}}

#### Alarm chất lượng RAG: `ragas-faithfulness-low`

```hcl
resource "aws_cloudwatch_metric_alarm" "ragas_faithfulness_low" {
  count = var.evaluation_image_pushed ? 1 : 0

  alarm_name          = "${local.name_prefix}-ragas-faithfulness-low"
  alarm_description   = "[Critical] Daily average RAGAS faithfulness score dropped below ${var.ragas_faithfulness_threshold} — answers may be losing grounding in retrieved context."
  namespace           = "RAGEvaluation"
  metric_name         = "Faithfulness"
  statistic           = "Average"
  period              = 86400 # one evaluation run/day
  evaluation_periods  = 1
  comparison_operator = "LessThanThreshold"
  threshold           = var.ragas_faithfulness_threshold
  treat_missing_data  = "notBreaching"

  alarm_actions = [var.alerts_critical_topic_arn]
  ok_actions    = [var.alerts_critical_topic_arn]

  tags = merge(var.tags, {
    Name     = "${local.name_prefix}-ragas-faithfulness-low"
    Severity = "Critical"
  })
}
```

{{% notice note %}}
📌 Alarm này **không nằm trong `modules/monitoring`** như 4 alarm ở Luồng 3, mà khai báo **ngay trong `modules/evaluation`** — vì nó phụ thuộc trực tiếp vào metric `RAGEvaluation/Faithfulness` do chính luồng này publish. Alarm trỏ ra `var.alerts_critical_topic_arn` — biến được **truyền từ module `monitoring` của Luồng 3 sang** module `evaluation` này, tái sử dụng đúng kênh SNS Critical đã có (xem lại [trang 5.5.1](../../5.5-Flow-3-Monitoring/5.5.1-SNS-2-kenh/)) thay vì tạo kênh cảnh báo riêng.
{{% /notice %}}

Ý nghĩa nghiệp vụ: nếu điểm **Faithfulness** trung bình hàng ngày tụt dưới ngưỡng, nghĩa là **câu trả lời của chatbot đang có dấu hiệu "bịa" thay vì bám vào tài liệu đã truy xuất** — đây là loại cảnh báo Critical vì ảnh hưởng trực tiếp tới độ tin cậy của cả hệ thống, không chỉ là vấn đề hạ tầng kỹ thuật thông thường.

---

#### Nội dung tiếp theo

- [5.6.3 - Logic đánh giá RAGAS (evaluation_runner.py)](../5.6.3-RAGAS-Evaluation-Logic/)
