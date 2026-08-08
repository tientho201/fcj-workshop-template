---
title: "SNS: 2 Channels By Severity"
date: 2026-08-03
weight: 1
chapter: false
pre: " <b> 5.5.1. </b> "
---

Monitoring infrastructure is declared in `modules/monitoring/main.tf`. This page covers the two SNS topics split by severity: Warning goes to email, Critical routes to Slack via AWS Chatbot — instead of dumping every alert into one channel (which causes alarm fatigue). Names use `local.name_prefix` (`${project_name}-${environment}`), e.g. `rag-app-dev-alerts-info`.

#### Alarm → SNS channel mapping

| Alarm                    | Severity | Destination               |
| ------------------------ | -------- | ------------------------- |
| `lambda-errors`          | Warning  | `alerts-info` → email     |
| `apigw-5xx`              | Warning  | `alerts-info` → email     |
| `bedrock-throttle`       | Critical | `alerts-critical` → Slack |
| `dlq-depth`              | Critical | `alerts-critical` → Slack |
| `ragas-faithfulness-low` | Critical | `alerts-critical` → Slack |

The first four alarms live in the monitoring module. `ragas-faithfulness-low` is declared in `modules/evaluation` but publishes to the same Critical topic. Thresholds and metrics are in [5.5.2](../5.5.2-CloudWatch-Alarms/).

#### alerts-info (Warning) — email

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
**Easy to miss:** after `terraform apply`, AWS emails a confirmation link to `var.alert_email`. The subscription stays **Pending** and **delivers no alerts** until that link is clicked. A successful apply does not mean the channel is live — after the first deploy, verify Confirmed status in the SNS Console.
{{% /notice %}}

#### alerts-critical (Critical) — Slack via AWS Chatbot

The Critical topic is always created. The AWS Chatbot configuration is only created when Slack is fully filled in — if left empty, alarms still publish to the topic but nothing reads them in Slack yet.

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

The IAM role attaches the managed `CloudWatchReadOnlyAccess` policy — AWS's recommendation so Chatbot can read alarm detail when rendering Slack messages, with no write permissions.

`local.slack_configured` rejects empty strings, the `"NONE"` sentinel, and `REQUIRED-TXXXXXXXX` / `REQUIRED-CXXXXXXXX` placeholders from `terraform.tfvars.example` — so applying an untouched example file does not fail mid-way.

In tfvars, set the **Slack channel ID** (`C0...`), not the channel name (e.g. `#rag-alerts`).

{{% notice warning %}}
**Manual prerequisite:** the Slack workspace must be OAuth-authorized with AWS Chatbot **in the Console first** (Chatbot → Configure new client → Slack). Terraform has no API for this OAuth step — skipping it and applying will fail looking for a valid `slack_team_id`. One of the few steps in the stack that cannot be fully automated with IaC.
{{% /notice %}}

#### PagerDuty (optional, off by default)

```hcl
# resource "aws_sns_topic_subscription" "alerts_critical_pagerduty" {
#   count     = var.enable_pagerduty ? 1 : 0
#   topic_arn = aws_sns_topic.alerts_critical.arn
#   protocol  = "https"
#   endpoint  = var.pagerduty_integration_url
# }
```

Left commented in code. To enable: uncomment, set `enable_pagerduty = true` and `pagerduty_integration_url` (Events API v2), then re-apply — no need to redesign routing.

---

#### Next topic

- [5.5.2 - CloudWatch Alarms](../5.5.2-CloudWatch-Alarms/)
