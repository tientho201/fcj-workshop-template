---
title: "CloudWatch Dashboard và Custom Metrics"
date: 2026-08-03
weight: 3
chapter: false
pre: " <b> 5.5.3. </b> "
---

Trang này trình bày CloudWatch Dashboard trong `modules/monitoring/main.tf` và 2 nhóm custom metric mà dashboard đọc thêm (cache hit rate từ chat-engine, điểm RAGAS từ evaluation-runner). Toàn bộ dashboard là một `aws_cloudwatch_dashboard` với `dashboard_body = jsonencode({ widgets = [...] })` — **9 widget** trên cùng một màn hình `…-overview`.

#### 9 widget của Dashboard

| #   | Widget                                  | Nguồn metric                                    |
| --- | --------------------------------------- | ----------------------------------------------- |
| 1   | Lambda Invocations / Errors             | `AWS/Lambda` (document-processor + chat-engine) |
| 2   | Lambda Duration (p50 / p99)             | `AWS/Lambda` Duration                           |
| 3   | API Gateway — Requests / 4xx / 5xx      | `AWS/ApiGateway` (+ dimension `Stage`)          |
| 4   | API Gateway Latency                     | Latency + IntegrationLatency                    |
| 5   | DynamoDB Capacity (on-demand, consumed) | Read/Write trên `chat_history` + `feedback`     |
| 6   | SQS Queue Depth (main + DLQs)           | ingestion-queue + 2 DLQ                         |
| 7   | Bedrock Invocation Count                | `AWS/Bedrock` Invocations                       |
| 8   | Semantic Cache Hit Rate                 | Custom EMF: `CacheHit` / `CacheMiss`            |
| 9   | RAGAS Evaluation Scores (daily)         | `RAGEvaluation` (put_metric_data)               |

Widget 1–7 đọc metric AWS tự sinh. Widget 8–9 **không do Terraform tạo metric** — Terraform chỉ khai báo widget đọc lại; dữ liệu chỉ xuất hiện khi Lambda/evaluation emit.

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

`local.cache_metric_namespace` lấy từ `modules/query` (`${name_prefix}/SemanticCache`) — producer và dashboard dùng cùng một giá trị, tránh lệch namespace.

#### Widget 8 — Cache Hit Rate (EMF từ chat-engine)

Sau mỗi request, chat-engine emit `CacheHit` hoặc `CacheMiss` (count = 1). Widget tính:

`IF((hit + miss) > 0, hit / (hit + miss) * 100, 0)`

```python
# modules/query/lambda_src/chat_engine/handler.py
def _emit_metric(name, value=1):

    print(
        json.dumps(
            {
                "_aws": {
                    "Timestamp": int(time.time() * 1000),
                    "CloudWatchMetrics": [
                        {
                            "Namespace": CACHE_METRIC_NAMESPACE,
                            "Dimensions": [[]],
                            "Metrics": [{"Name": name, "Unit": "Count"}],
                        }
                    ],
                },
                name: value,
            }
        )
    )


```

{{% notice tip %}}
**Vì sao EMF (log) thay vì `PutMetricData`?** `chat-engine` chạy trong VPC **không có NAT Gateway**. Gọi `cloudwatch:PutMetricData` sẽ cần thêm interface VPC endpoint `monitoring` (chi phí theo giờ) + quyền IAM. EMF tận dụng đường ghi log Lambda vốn đã có — CloudWatch tự parse metric, không thêm endpoint.
{{% /notice %}}

#### Widget 9 — RAGAS scores (put_metric_data từ evaluation-runner)

Evaluation-runner (Luồng 4) chạy theo lịch ngày, gọi `put_metric_data` vào namespace `RAGEvaluation`. Lambda evaluation **không** nằm trong VPC chat-engine nên dùng API CloudWatch trực tiếp được.

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

Widget RAGAS dùng `period = 86400` (1 ngày). **Trống cho đến khi evaluation chạy ít nhất một lần** (`evaluation_image_pushed = true` và scheduler/manual invoke) — không phải lỗi cấu hình dashboard.

---

#### Nội dung tiếp theo

- [5.5.4 - Kiểm thử end-to-end](../5.5.4-End-to-End-Testing/)
