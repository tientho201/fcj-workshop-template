---
title: "CloudWatch Alarms"
date: 2026-08-03
weight: 2
chapter: false
pre: " <b> 5.5.2. </b> "
---

This page covers the 4 CloudWatch Alarms in `modules/monitoring/main.tf` — each wired to one SNS topic from [5.5.1](../5.5.1-SNS-2-Channels-By-Severity/). A fifth alarm (`ragas-faithfulness-low`) lives in `modules/evaluation` and is documented under Stream 4.

| Alarm              | How it is measured                                                                          | Default threshold | SNS               |
| ------------------ | ------------------------------------------------------------------------------------------- | ----------------- | ----------------- |
| `lambda-errors`    | Sum of `Errors` across document-processor + chat-engine (+ evaluation-runner once deployed) | > 5 / 5 min       | `alerts-info`     |
| `apigw-5xx`        | `(5XXError / Count) * 100`, wrapped in `IF(requests > 0, …)`                                | > 5% / 5 min      | `alerts-info`     |
| `bedrock-throttle` | Log metric filters scanning `"ThrottlingException"` on both Lambdas                         | > 3 / 5 min       | `alerts-critical` |
| `dlq-depth`        | Sum of `ApproximateNumberOfMessagesVisible` on both DLQs                                    | > 0               | `alerts-critical` |

Every alarm sets `treat_missing_data = "notBreaching"` — no traffic means no false alarm. Thresholds come from variables (`lambda_error_alarm_threshold`, `apigw_5xx_threshold_percent`, `bedrock_throttle_alarm_threshold`).

#### lambda-errors and apigw-5xx (Warning)

```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_errors" {
  alarm_name          = "${local.name_prefix}-lambda-errors"
  alarm_description   = "[Warning] Combined Lambda Errors … > ${var.lambda_error_alarm_threshold} in 5 minutes."
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.lambda_error_alarm_threshold
  evaluation_periods  = 1
  treat_missing_data  = "notBreaching"

  metric_query {
    id          = "doc_processor_errors"
    return_data = false
    metric {
      metric_name = "Errors"
      namespace   = "AWS/Lambda"
      period      = 300
      stat        = "Sum"
      dimensions  = { FunctionName = var.document_processor_function_name }
    }
  }

  metric_query {
    id          = "chat_engine_errors"
    return_data = false
    metric {
      metric_name = "Errors"
      namespace   = "AWS/Lambda"
      period      = 300
      stat        = "Sum"
      dimensions  = { FunctionName = var.chat_engine_function_name }
    }
  }

  # dynamic "metric_query" for evaluation-runner when local.evaluation_lambda_present

  metric_query {
    id          = "combined"
    expression  = local.evaluation_lambda_present
      ? "doc_processor_errors + chat_engine_errors + eval_runner_errors"
      : "doc_processor_errors + chat_engine_errors"
    label       = "CombinedLambdaErrors"
    return_data = true
  }

  alarm_actions = [aws_sns_topic.alerts_info.arn]
  ok_actions    = [aws_sns_topic.alerts_info.arn]
}

resource "aws_cloudwatch_metric_alarm" "apigw_5xx" {
  alarm_name          = "${local.name_prefix}-apigw-5xx"
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.apigw_5xx_threshold_percent
  evaluation_periods  = 1
  treat_missing_data  = "notBreaching"

  metric_query {
    id          = "requests"
    return_data = false
    metric {
      metric_name = "Count"
      namespace   = "AWS/ApiGateway"
      period      = 300
      stat        = "Sum"
      dimensions = {
        ApiName = var.api_gateway_name
        Stage   = var.api_gateway_stage_name
      }
    }
  }

  metric_query {
    id          = "errors_5xx"
    return_data = false
    metric {
      metric_name = "5XXError"
      namespace   = "AWS/ApiGateway"
      period      = 300
      stat        = "Sum"
      dimensions = {
        ApiName = var.api_gateway_name
        Stage   = var.api_gateway_stage_name
      }
    }
  }

  metric_query {
    id          = "error_rate_pct"
    expression  = "IF(requests > 0, (errors_5xx / requests) * 100, 0)"
    label       = "5xxErrorRatePercent"
    return_data = true
  }

  alarm_actions = [aws_sns_topic.alerts_info.arn]
  ok_actions    = [aws_sns_topic.alerts_info.arn]
}
```

`lambda-errors` uses a `dynamic "metric_query"` — evaluation-runner is only included when `var.evaluation_function_name` is set (after the evaluation image is pushed). Monitoring can be applied before evaluation without breaking the alarm on a missing function name.

`apigw-5xx` dimensions include both `ApiName` and `Stage`. The `IF(requests > 0, …)` expression avoids divide-by-zero when there is no traffic off-hours.

{{% notice tip %}}
The alarm uses an **error rate %** instead of an absolute 5xx count: low demo traffic makes a few errors too noisy as a fixed count; higher traffic later makes a fixed count too loose. A rate is more stable across environments.
{{% /notice %}}

#### bedrock-throttle (Critical) — log scan instead of native Bedrock metrics

Bedrock exposes a native `AWS/Bedrock` / `InvocationThrottles` metric. This stack takes another path: two log metric filters scan for `"ThrottlingException"` on the document-processor and chat-engine log groups. Chat-engine calls `logger.exception(...)` on throttle (returns HTTP 429) — the traceback still carries the exception name for the filter to match. The Terraform comment notes you can swap to the native metric later if you want to stop depending on log text.

```hcl
resource "aws_cloudwatch_log_metric_filter" "bedrock_throttle_document_processor" {
  name           = "${local.name_prefix}-bedrock-throttle-document-processor"
  log_group_name = var.document_processor_log_group_name
  pattern        = "\"ThrottlingException\""

  metric_transformation {
    name          = local.bedrock_metric_name      # BedrockThrottlingExceptions
    namespace     = local.bedrock_metric_namespace # ${name_prefix}/Bedrock
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_log_metric_filter" "bedrock_throttle_chat_engine" {
  name           = "${local.name_prefix}-bedrock-throttle-chat-engine"
  log_group_name = var.chat_engine_log_group_name
  pattern        = "\"ThrottlingException\""

  metric_transformation {
    name          = local.bedrock_metric_name
    namespace     = local.bedrock_metric_namespace
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "bedrock_throttle" {
  alarm_name          = "${local.name_prefix}-bedrock-throttle"
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.bedrock_throttle_alarm_threshold
  evaluation_periods  = 1
  period              = 300
  statistic           = "Sum"
  metric_name         = local.bedrock_metric_name
  namespace           = local.bedrock_metric_namespace
  treat_missing_data  = "notBreaching"

  alarm_actions = [aws_sns_topic.alerts_critical.arn]
  ok_actions    = [aws_sns_topic.alerts_critical.arn]
}
```

Both filters write to **one** custom metric (`BedrockThrottlingExceptions` in a namespace like `rag-app-dev/Bedrock`). The alarm only needs `Sum` to combine throttles from both Lambdas.

Chat-side throttle handling: [5.4.6 Error Handling](../../5.4-Realtime-QA/5.4.6-Error-Handling-OCR-Decision/).

#### dlq-depth (Critical) — both DLQ layers summed

```hcl
resource "aws_cloudwatch_metric_alarm" "dlq_depth" {
  alarm_name          = "${local.name_prefix}-dlq-depth"
  comparison_operator = "GreaterThanThreshold"
  threshold           = 0
  evaluation_periods  = 1
  treat_missing_data  = "notBreaching"

  metric_query {
    id          = "main_dlq"
    return_data = false
    metric {
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      period      = 300
      stat        = "Maximum"
      dimensions  = { QueueName = var.ingestion_dlq_name }
    }
  }

  metric_query {
    id          = "fn_dlq"
    return_data = false
    metric {
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      period      = 300
      stat        = "Maximum"
      dimensions  = { QueueName = var.document_processor_fn_dlq_name }
    }
  }

  metric_query {
    id          = "total"
    expression  = "main_dlq + fn_dlq"
    label       = "TotalDlqDepth"
    return_data = true
  }

  alarm_actions = [aws_sns_topic.alerts_critical.arn]
  ok_actions    = [aws_sns_topic.alerts_critical.arn]
}
```

The two DLQ layers are described in [5.3.1 Infrastructure S3/SQS](../../5.3-Data-Ingestion/5.3.1-Infrastructure-S3-SQS/). Threshold `> 0`: a healthy run leaves DLQs empty — a single message is enough for Critical. Each queue uses `Maximum`, then they are summed, so a short spike is not washed out by an Average.

#### Fifth alarm (Stream 4)

`ragas-faithfulness-low` — daily Faithfulness average `< 0.7`, period 86400s — also publishes to `alerts-critical`, but the resource lives in the evaluation module and is only created when `evaluation_image_pushed = true`. Kept out of this page so the four alarms that exist as soon as monitoring is applied stay clear.

---

#### Next topic

- [5.5.3 - CloudWatch Dashboard và Custom Metrics](../5.5.3-Dashboard-Custom-Metrics/)
