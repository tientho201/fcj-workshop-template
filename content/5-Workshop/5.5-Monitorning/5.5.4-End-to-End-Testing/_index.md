---
title: "End-to-End Testing"
date: 2026-08-03
weight: 4
chapter: false
pre: " <b> 5.5.4. </b> "
---

After SNS ([5.5.1](../5.5.1-SNS-2-Channels-By-Severity/)), Alarms ([5.5.2](../5.5.2-CloudWatch-Alarms/)), and Dashboard ([5.5.3](../5.5.3-Dashboard-Custom-Metrics/)), the last step is confirming the alert chain actually works — often skipped because infrastructure "looks fine" even when the email subscription is still Pending or Chatbot is not attached to Slack.

#### Test scenarios

| #   | Scenario                                                                                  | Expected result                                                                                                       |
| --- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| 1   | SNS Console → `alerts-info` subscription                                                  | Status `Confirmed` (not `Pending confirmation`)                                                                       |
| 2   | AWS Chatbot → Slack channel configuration                                                 | Correct workspace/channel; if `slack_*` is `NONE`, the Chatbot resource does not exist (by design)                    |
| 3   | Simulate Lambda failures (throw exceptions for several minutes past the Errors threshold) | `lambda-errors` → `ALARM`, email via `alerts-info`                                                                    |
| 4   | Cause API/Lambda 5xx (e.g. a transient fault in chat-engine returning 500)                | `apigw-5xx` → `ALARM` when 5xx rate exceeds the % threshold, email received                                           |
| 5   | Trigger `ThrottlingException` (burst Bedrock calls or mock the error in code)             | Log filter matches, `bedrock-throttle` → `ALARM`, Slack message via `alerts-critical`                                 |
| 6   | Push one failing message into a DLQ (ingestion DLQ or function DLQ)                       | `dlq-depth` → `ALARM` as soon as depth > 0, Slack message                                                             |
| 7   | Open dashboard `${name_prefix}-overview`                                                  | 7 AWS widgets populate after traffic; Cache Hit Rate after a few `/chat` calls; RAGAS after evaluation has run ≥ once |
| 8   | Stop the fault simulation after `ALARM`                                                   | Alarm returns to `OK` on its own once the threshold is no longer breached — no manual reset                           |

{{% notice tip %}}
Scenario 4: a wrong path/method usually yields **4xx**, which does not trip `apigw-5xx`. You need a server-side failure (5xx) or a high enough 5xx rate in the 5-minute window.
{{% /notice %}}

#### Outcomes

- Two SNS channels by severity — Warning (email) and Critical (Slack) — reduce alarm fatigue.
- `bedrock-throttle` reuses Lambda logs (Stream 2) instead of a separate throttle pipeline.
- `dlq-depth` covers both DLQ layers (Stream 1) — no silent backlog on one tier.
- Dashboard with 9 widgets: 7 AWS metrics + cache hit rate (EMF) + RAGAS (`put_metric_data`).
- `treat_missing_data = "notBreaching"` avoids false alarms in low-traffic environments.
