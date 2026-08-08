---
title: "SNS: 2 kênh theo mức độ nghiêm trọng"
date: 2026-08-03
weight: 1
chapter: false
pre: " <b> 5.5.1. </b> "
---

Hạ tầng giám sát được khai báo tại `modules/monitoring/main.tf`. Trang này trình bày 2 SNS topic tách theo mức độ nghiêm trọng: Warning gửi email, Critical route sang Slack qua AWS Chatbot — thay vì gộp mọi cảnh báo vào một kênh (tránh alarm fatigue). Naming dùng `local.name_prefix` (`${project_name}-${environment}`), ví dụ `rag-app-dev-alerts-info`.

#### Mapping alarm → kênh SNS

| Alarm                    | Severity | Đích                      |
| ------------------------ | -------- | ------------------------- |
| `lambda-errors`          | Warning  | `alerts-info` → email     |
| `apigw-5xx`              | Warning  | `alerts-info` → email     |
| `bedrock-throttle`       | Critical | `alerts-critical` → Slack |
| `dlq-depth`              | Critical | `alerts-critical` → Slack |
| `ragas-faithfulness-low` | Critical | `alerts-critical` → Slack |

Bốn alarm đầu nằm trong monitoring module. `ragas-faithfulness-low` khai báo ở `modules/evaluation` nhưng publish vào cùng topic Critical. Chi tiết ngưỡng và metric ở [5.5.2](../5.5.2-CloudWatch-Alarms/).

#### alerts-info (Warning) — gửi email

```hcl
resource "aws_sns_topic" "alerts_info" {
  name = "${local.name_prefix}-alerts-info"

  tags = merge(var.tags, {
    Name     = "${local.name_prefix}-alerts-info"
    Severity = "Warning"
  })
}

resource "aws_sns_topic_subscription" "alerts_info_email" {
  topic_arn = aws_sns_topic.alerts_info.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

{{% notice warning %}}
**Điểm dễ vấp:** sau `terraform apply`, AWS gửi email xác nhận tới `var.alert_email`. Subscription ở trạng thái **Pending** và **không gửi cảnh báo nào** cho đến khi bấm link xác nhận. Apply thành công không có nghĩa kênh đã hoạt động — sau lần deploy đầu, kiểm tra trạng thái Confirmed trên SNS Console.
{{% /notice %}}

#### alerts-critical (Critical) — Slack qua AWS Chatbot

Topic Critical luôn được tạo. Cấu hình AWS Chatbot chỉ tạo khi Slack đã điền đủ — nếu bỏ trống, alarm vẫn publish vào topic nhưng chưa có subscriber trên Slack.

```hcl
resource "aws_sns_topic" "alerts_critical" {
  name = "${local.name_prefix}-alerts-critical"

  tags = merge(var.tags, {
    Name     = "${local.name_prefix}-alerts-critical"
    Severity = "Critical"
  })
}

data "aws_iam_policy_document" "chatbot_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["chatbot.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "chatbot" {
  name               = "${local.name_prefix}-chatbot-role"
  assume_role_policy = data.aws_iam_policy_document.chatbot_assume.json

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "chatbot_cloudwatch_readonly" {
  role       = aws_iam_role.chatbot.name
  policy_arn = "arn:${data.aws_partition.current.partition}:iam::aws:policy/CloudWatchReadOnlyAccess"
}

resource "aws_chatbot_slack_channel_configuration" "critical" {
  count              = local.slack_configured ? 1 : 0
  configuration_name = "${local.name_prefix}-critical-alerts"
  iam_role_arn       = aws_iam_role.chatbot.arn
  slack_channel_id   = var.slack_channel_id
  slack_team_id      = var.slack_workspace_id
  sns_topic_arns     = [aws_sns_topic.alerts_critical.arn]
  logging_level      = "ERROR"

  tags = var.tags
}
```

IAM role gắn managed policy `CloudWatchReadOnlyAccess` — recommendation của AWS để Chatbot đọc chi tiết alarm khi render tin nhắn Slack, không cần quyền ghi.

`local.slack_configured` loại chuỗi rỗng, sentinel `"NONE"`, và placeholder `REQUIRED-TXXXXXXXX` / `REQUIRED-CXXXXXXXX` trong `terraform.tfvars.example` — tránh apply với file example chưa sửa rồi fail giữa chừng.

Trong tfvars cần điền **Slack channel ID** (`C0...`), không phải tên channel (ví dụ `#rag-alerts`).

{{% notice warning %}}
**Tiền điều kiện thủ công:** workspace Slack phải authorize OAuth với AWS Chatbot **trên Console trước** (Chatbot → Configure new client → Slack). Terraform không có API cho bước OAuth này — bỏ qua rồi apply sẽ fail vì không tìm thấy `slack_team_id` hợp lệ. Đây là một trong số ít bước không tự động hóa hoàn toàn bằng IaC trong stack.
{{% /notice %}}

#### PagerDuty (tùy chọn, chưa bật mặc định)

```hcl
# resource "aws_sns_topic_subscription" "alerts_critical_pagerduty" {
#   count     = var.enable_pagerduty ? 1 : 0
#   topic_arn = aws_sns_topic.alerts_critical.arn
#   protocol  = "https"
#   endpoint  = var.pagerduty_integration_url
# }
```

Tích hợp để sẵn dạng comment. Bật khi cần: uncomment, set `enable_pagerduty = true` và `pagerduty_integration_url` (Events API v2), rồi re-apply — không phải thiết kế lại routing.

---

#### Nội dung tiếp theo

- [5.5.2 - CloudWatch Alarms](../5.5.2-CloudWatch-Alarms/)
