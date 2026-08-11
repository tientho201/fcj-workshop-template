---
title: "IAM Permissions and RAG Quality Alarm"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.6.2 </b> "
---

#### IAM Role for `evaluation_runner`

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

6 permission groups:

| Permission                          | Scope                                                                               | Notes                                                        |
| ----------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `sts:AssumeRole`                    | Service `lambda.amazonaws.com`                                                      | Allows the AWS Lambda service to assume this IAM Role        |
| `logs:CreateLogStream/PutLogEvents` | Log Group `/aws/lambda/${local.evaluation_fn_name}`                                 | Writes Lambda execution logs to CloudWatch Logs              |
| `dynamodb:Scan/Query/GetItem`       | `feedback_table` + `chat_history_table` (2 tables from Stream 2)                    | Reads chat history and user feedback data for evaluation     |
| `s3:PutObject`                      | Specifically `evaluation_results` bucket                                            | Writes detailed evaluation result files (JSON/CSV)           |
| `bedrock:InvokeModel`               | Scoped to exactly 1 model (`ragas_judge_model_id`)                                  | Allows invoking Bedrock LLM as the "judge" for RAGAS scoring |
| `cloudwatch:PutMetricData`          | `resources = "*"` **but** with condition `cloudwatch:namespace = ["RAGEvaluation"]` | Pushes RAGAS score metrics to CloudWatch Custom Metrics      |

{{% notice tip %}}
**Why must `cloudwatch:PutMetricData` use `resources = "*"`?** This is one of the few AWS actions that **does not support scoping by ARN** — the action itself has no concept of a specific "resource" to restrict. Instead, the project uses an **IAM Condition** constraining `cloudwatch:namespace = RAGEvaluation` — meaning this Lambda **is only allowed to write metrics into the exact namespace `RAGEvaluation`**, and cannot write to any other namespace (including AWS system namespaces). This remains true least-privilege, just using a different restriction mechanism than other actions (using `Condition` instead of `Resource`).
{{% /notice %}}

#### RAG Quality Alarm: `ragas-faithfulness-low`

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
📌 This alarm **is not located in `modules/monitoring`** like the 4 alarms in Stream 3, but declared **directly inside `modules/evaluation`** — because it directly depends on the `RAGEvaluation/Faithfulness` metric published by this stream itself. The alarm points to `var.alerts_critical_topic_arn` — a variable **passed from Stream 3's `monitoring` module into** this `evaluation` module, reusing the existing Critical SNS channel (refer back to [page 5.5.1](../../5.5-Monitoring/5.5.1-SNS-2-Channels-By-Severity/)) instead of creating a separate notification channel.
{{% /notice %}}

Business significance: If the daily average **Faithfulness** score drops below the threshold, it means **the chatbot's answers show signs of "hallucination" rather than grounding in retrieved documents** — this is a Critical alert because it directly impacts the trustworthiness of the entire system, not just a routine technical infrastructure issue.

---

#### Next Content

- [5.6.3 - RAGAS Evaluation Logic (evaluation_runner.py)](../5.6.3-RAGAS-Evaluation-Logic/)
