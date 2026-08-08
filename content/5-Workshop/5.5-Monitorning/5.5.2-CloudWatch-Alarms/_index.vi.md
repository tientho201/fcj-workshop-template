---
title: "CloudWatch Alarms"
date: 2026-08-03
weight: 2
chapter: false
pre: " <b> 5.5.2. </b> "
---

Trang này trình bày 4 CloudWatch Alarm trong `modules/monitoring/main.tf` — mỗi alarm nối đúng 1 SNS topic đã tạo ở [5.5.1](../5.5.1-SNS-2-Channels-By-Severity/). Alarm thứ 5 (`ragas-faithfulness-low`) nằm ở `modules/evaluation` và được mô tả ở Luồng 4.

| Alarm              | Cách đo                                                                                | Ngưỡng mặc định | SNS               |
| ------------------ | -------------------------------------------------------------------------------------- | --------------- | ----------------- |
| `lambda-errors`    | Cộng `Errors` của document-processor + chat-engine (+ evaluation-runner nếu đã deploy) | > 5 / 5 phút    | `alerts-info`     |
| `apigw-5xx`        | `(5XXError / Count) * 100`, bọc `IF(requests > 0, …)`                                  | > 5% / 5 phút   | `alerts-info`     |
| `bedrock-throttle` | Log metric filter quét `"ThrottlingException"` từ 2 Lambda                             | > 3 / 5 phút    | `alerts-critical` |
| `dlq-depth`        | Cộng `ApproximateNumberOfMessagesVisible` của 2 DLQ                                    | > 0             | `alerts-critical` |

Tất cả đặt `treat_missing_data = "notBreaching"` — không có traffic thì không báo động giả. Ngưỡng lấy từ biến (`lambda_error_alarm_threshold`, `apigw_5xx_threshold_percent`, `bedrock_throttle_alarm_threshold`).

#### lambda-errors và apigw-5xx (Warning)

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

  # dynamic "metric_query" cho evaluation-runner khi local.evaluation_lambda_present

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

`lambda-errors` dùng `dynamic "metric_query"` — evaluation-runner chỉ được cộng khi `var.evaluation_function_name` đã có (sau khi push image evaluation). Có thể apply monitoring trước, evaluation sau, mà alarm không vỡ vì thiếu function name.

`apigw-5xx` dimensions gồm cả `ApiName` và `Stage`. Biểu thức `IF(requests > 0, …)` tránh chia cho 0 khi ngoài giờ không có traffic.

{{% notice tip %}}
Alarm dùng **tỷ lệ %** thay vì số tuyệt đối 5xx: với traffic demo thấp, vài lỗi dễ làm nhiễu; traffic cao hơn thì ngưỡng tuyệt đối lại quá lỏng. Rate ổn định hơn giữa các môi trường.
{{% /notice %}}

#### bedrock-throttle (Critical) — quét log thay vì metric Bedrock gốc

Bedrock có metric native `AWS/Bedrock` / `InvocationThrottles`. Stack này chọn cách khác: 2 log metric filter quét chuỗi `"ThrottlingException"` trên log group của document-processor và chat-engine. Chat-engine gọi `logger.exception(...)` khi bắt throttle (trả HTTP 429) — traceback vẫn giữ tên exception để filter khớp được. Comment trong Terraform ghi sẵn hướng đổi sang metric native nếu muốn bỏ phụ thuộc log text.

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

Hai filter ghi **cùng một** custom metric (`BedrockThrottlingExceptions` trong namespace kiểu `rag-app-dev/Bedrock`). Alarm chỉ cần `Sum` để gộp throttle từ cả hai Lambda.

Logic bắt throttle phía chat: [5.4.6 Error Handling](../../5.4-Realtime-QA/5.4.6-Error-Handling-OCR-Decision/).

#### dlq-depth (Critical) — cộng cả 2 tầng DLQ

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

Hai tầng DLQ đã mô tả ở [5.3.1 Infrastructure S3/SQS](../../5.3-Data-Ingestion/5.3.1-Infrastructure-S3-SQS/). Ngưỡng `> 0`: vận hành bình thường DLQ phải trống — chỉ cần 1 message cũng đủ Critical. Dùng `Maximum` trên từng queue rồi cộng, tránh bỏ sót spike nếu chỉ nhìn Average.

#### Alarm thứ 5 (Luồng 4)

`ragas-faithfulness-low` — Faithfulness trung bình ngày `< 0.7`, period 86400s — cũng publish `alerts-critical`, nhưng resource nằm ở evaluation module và chỉ tạo khi `evaluation_image_pushed = true`. Không gộp vào trang này để tách rõ 4 alarm luôn có sau khi apply monitoring.

---

#### Nội dung tiếp theo

- [5.5.3 - CloudWatch Dashboard và Custom Metrics](../5.5.3-Dashboard-Custom-Metrics/)
