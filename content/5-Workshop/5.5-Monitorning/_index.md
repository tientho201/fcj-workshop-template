---
title: "Monitoring and Alerting"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

#### Introduction

Stream 3 watches the health of Streams 1 and 2 in real time, classifies alerts by severity, and routes them to the right operational channel (email or Slack). A fifth quality alarm from Stream 4 (`ragas-faithfulness-low`) also publishes into the same Critical SNS topic — without a separate alerting stack.

The stream is declared in `modules/monitoring/main.tf` and consists of 2 main parts:

- **Infrastructure (Terraform)** — 2 SNS topics by severity (`alerts-info` → email, `alerts-critical` → Slack via AWS Chatbot), 4 CloudWatch Alarms in the monitoring module (plus the RAGAS alarm owned by `modules/evaluation`), and 1 CloudWatch Dashboard with 9 widgets.
- **Custom metrics (application-emitted)** — chat-engine emits cache hit/miss via EMF; evaluation-runner publishes RAGAS scores via `put_metric_data`. Terraform only wires widgets/alarms that read these metrics; it does not create the metric series themselves.
  {{% notice note %}}
  📌 Stream 3 **is not a separate monitoring product**. It assembles what AWS/Lambda already produce (native metrics + logs) into structured alarms and a dashboard, then adds a few custom series only where AWS has no built-in equivalent (cache hit rate, RAGAS scores). `bedrock-throttle` is the special case: it uses log metric filters on `"ThrottlingException"` instead of a native Bedrock throttle metric.
  {{% /notice %}}

#### Data Flow Diagram

![Detailed Diagram for Stream 3 - Monitoring and Alerting](/images/5-Workshop/5.5-Monitorning/image.png)

#### Detailed Contents

1. [SNS: 2 Channels by Severity](5.5.1-SNS-2-Channels-By-Severity/)
2. [CloudWatch Alarms](5.5.2-CloudWatch-Alarms/)
3. [CloudWatch Dashboard and Custom Metrics](5.5.3-Dashboard-Custom-Metrics/)
4. [End-to-End Testing](5.5.4-End-to-End-Testing/)
