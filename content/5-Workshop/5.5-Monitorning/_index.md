---
title: "Monitoring and Alerting"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

#### Introduction

Stream 3 monitors the entire system in real-time, categorizes alerts by severity, and sends notifications to the appropriate operational channels. Infrastructure is declared in `modules/monitoring/main.tf`.

{{% notice note %}}
📌 Stream 3 **is not a separate monitoring system**, but rather assembles what AWS/Lambda automatically generates (native metrics + logs) into structured alarms/dashboards, plus a few custom metrics logged by the application when AWS does not provide them out of the box (cache hit rate, RAGAS score).
{{% /notice %}}

The stream consists of 3 main parts:

- **2 SNS channels by severity** — `alerts-info` (Warning, sent via email) and `alerts-critical` (Critical, routed to Slack via AWS Chatbot).
- **4 CloudWatch Alarms** — each alarm connects to exactly 1 of the 2 channels above, where `bedrock-throttle` is a special alarm scanning logs instead of using native metrics.
- **1 CloudWatch Dashboard with 8 widgets** — 6 widgets read built-in AWS metrics, and the last 2 widgets read custom metrics logged directly by Lambda (EMF).

#### Data Flow Diagram

![Detailed Diagram for Stream 3 - Monitoring and Alerting](/images/5-Workshop/5.5-Monitorning/image.png)

#### Detailed Contents

1. [SNS: 2 Channels by Severity](5.5.1-SNS-2-Channels-By-Severity/)
2. [CloudWatch Alarms](5.5.2-CloudWatch-Alarms/)
3. [CloudWatch Dashboards and Custom Metrics](5.5.3-Dashboard-Custom-Metrics/)
4. [End-to-End Testing](5.5.4-End-to-End-Testing/)
