---
title: "CloudWatch Dashboard and Custom Metrics"
date: 2026-08-03
weight: 3
chapter: false
pre: " <b> 5.5.3. </b> "
---

This page covers the CloudWatch Dashboard in `modules/monitoring/main.tf` and the two custom-metric groups the dashboard also reads (cache hit rate from chat-engine, RAGAS scores from evaluation-runner). The dashboard is a single `aws_cloudwatch_dashboard` with `dashboard_body = jsonencode({ widgets = [...] })` — **9 widgets** on one `${…}-overview` screen.

#### The 9 dashboard widgets

| #   | Widget                                  | Metric source                                   |
| --- | --------------------------------------- | ----------------------------------------------- |
| 1   | Lambda Invocations / Errors             | `AWS/Lambda` (document-processor + chat-engine) |
| 2   | Lambda Duration (p50 / p99)             | `AWS/Lambda` Duration                           |
| 3   | API Gateway — Requests / 4xx / 5xx      | `AWS/ApiGateway` (includes `Stage` dimension)   |
| 4   | API Gateway Latency                     | Latency + IntegrationLatency                    |
| 5   | DynamoDB Capacity (on-demand, consumed) | Read/Write on `chat_history` + `feedback`       |
| 6   | SQS Queue Depth (main + DLQs)           | ingestion-queue + both DLQs                     |
| 7   | Bedrock Invocation Count                | `AWS/Bedrock` Invocations                       |
| 8   | Semantic Cache Hit Rate                 | Custom EMF: `CacheHit` / `CacheMiss`            |
| 9   | RAGAS Evaluation Scores (daily)         | `RAGEvaluation` (`put_metric_data`)             |

Widgets 1–7 read AWS-built-in metrics. Widgets 8–9 **do not create metrics in Terraform** — Terraform only declares widgets that read them; data appears only after the Lambda/evaluation job emits.

```hcl
resource "aws_cloudwatch_dashboard" "this" {
  dashboard_name = "${local.name_prefix}-overview"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 12
        y      = 18
        width  = 12
        height = 6
        properties = {
          title  = "Semantic Cache Hit Rate"
          region = data.aws_region.current.name
          metrics = [
            [{ expression = "IF((m1+m2)>0, (m1/(m1+m2))*100, 0)", label = "Cache Hit Rate %", id = "hit_rate_pct" }],
            [local.cache_metric_namespace, "CacheHit", { stat = "Sum", id = "m1", visible = false }],
            [local.cache_metric_namespace, "CacheMiss", { stat = "Sum", id = "m2", visible = false }],
          ]
        }
      },

      {
        type   = "metric"
        x      = 0
        y      = 24
        width  = 24
        height = 6
        properties = {
          title  = "RAGAS Evaluation Scores (daily)"
          region = data.aws_region.current.name
          period = 86400
          metrics = [
            ["RAGEvaluation", "Faithfulness", { stat = "Average", label = "Faithfulness" }],
            ["RAGEvaluation", "Answer_relevancy", { stat = "Average", label = "Answer Relevancy" }],
            ["RAGEvaluation", "Context_precision", { stat = "Average", label = "Context Precision" }],
            ["RAGEvaluation", "Context_recall", { stat = "Average", label = "Context Recall" }],
          ]
        }
      },
    ]
  })
}
```

`local.cache_metric_namespace` comes from `modules/query` (`${name_prefix}/SemanticCache`) — producer and dashboard share one value so namespaces cannot drift.

#### Widget 8 — Cache Hit Rate (EMF from chat-engine)

On every request, chat-engine emits `CacheHit` or `CacheMiss` (count = 1). The widget computes:

`IF((hit + miss) > 0, hit / (hit + miss) * 100, 0)`

```python
# modules/query/lambda_src/chat_engine/handler.py
def _emit_metric(name, value=1):
    """EMF: structured log line — CloudWatch Logs parses it into a metric."""
    print(
        json.dumps(
            {
                "_aws": {
                    "Timestamp": int(time.time() * 1000),
                    "CloudWatchMetrics": [
                        {
                            "Namespace": CACHE_METRIC_NAMESPACE,  # ${name_prefix}/SemanticCache
                            "Dimensions": [[]],
                            "Metrics": [{"Name": name, "Unit": "Count"}],
                        }
                    ],
                },
                name: value,
            }
        )
    )

# call: _emit_metric("CacheHit") or _emit_metric("CacheMiss")
```

{{% notice tip %}}
**Why EMF (logs) instead of `PutMetricData`?** `chat-engine` runs in a VPC **with no NAT Gateway**. Calling `cloudwatch:PutMetricData` would need a `monitoring` interface VPC endpoint (hourly cost) plus IAM. EMF reuses the Lambda log path that already works — CloudWatch parses the metric with no extra endpoint.
{{% /notice %}}

#### Widget 9 — RAGAS scores (`put_metric_data` from evaluation-runner)

The evaluation-runner (Stream 4) runs on a daily schedule and calls `put_metric_data` into namespace `RAGEvaluation`. That Lambda is **not** in the chat-engine VPC, so the CloudWatch API is available directly.

```python
# docker/evaluation-runner/evaluation_runner.py
cloudwatch.put_metric_data(
    Namespace="RAGEvaluation",
    MetricData=[
        {"MetricName": name.capitalize(), "Value": value, "Unit": "None"}
        for name, value in averages.items()
    ],
)
# averages keys: faithfulness, answer_relevancy, context_precision, context_recall
# → MetricName: Faithfulness, Answer_relevancy, …
```

The RAGAS widget uses `period = 86400` (one day). It stays **empty until evaluation has run at least once** (`evaluation_image_pushed = true` and scheduler/manual invoke) — that is expected, not a dashboard misconfiguration.

---

#### Next content

- [5.5.4 - End-to-End Testing](../5.5.4-End-to-End-Testing/)
