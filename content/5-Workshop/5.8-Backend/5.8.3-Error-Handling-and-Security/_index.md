---
title: "Error Handling & Security"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.8.3 </b> "
---

#### IAM Principles Across Both Lambdas

No policy uses `Resource: "*"` **unless forced by AWS APIs** (Textract, X-Ray, VPC ENI management, `cloudwatch:PutMetricData`) — and **every exception includes a comment explaining why**, plus a `condition` to narrow the scope if the API permits (e.g. `PutMetricData` is strictly locked to the exact `RAGEvaluation` namespace using the `cloudwatch:namespace` condition — refer back to [Stream 4, page 5.6.2](../../5.6-RAGAS/5.6.2-IAM-Alarm-RAGAS/)).

{{% notice tip %}}
**AWS managed policies** (`AWSLambdaBasicExecutionRole`, `AWSLambdaVPCAccessExecutionRole`) **were intentionally avoided** because they come with wider scope than necessary (`logs:*` on **every** log group in the account) — instead, each Lambda has a custom-written policy, scoped to exactly **its own log group**.
{{% /notice %}}

#### Recurring Pattern: Base Table Permissions Do Not Automatically Include GSI Permissions

{{% notice warning %}}
📌 **This is an IAM pattern appearing in at least 2 places in the project**, worth noting as a general rule: `evaluation_runner` requires **1 separate IAM statement** to `Query` the `message_id-index` GSI (`"${chat_history_table_arn}/index/*"`) — `Query`/`Scan`/`GetItem` permissions granted on the `chat_history` base table itself **do not automatically allow Query on its index** (full details on [Stream 4, page 5.6.2](../../5.6-RAGAS/5.6.2-IAM-Alarm-RAGAS/)).

**This exact pattern recurs in `modules/ingestion/main.tf`** for the `document-id-index` GSI of the `child_chunks` table — used to delete all old chunks by `document_id` during re-indexing (refer back to [Stream 1, page 5.3.4](../../5.3-Data-Ingestion/5.3.4-Chunking-Embedding/)). `document_processor` also requires a separate IAM statement for this GSI, distinct from permissions on the base `child_chunks` table.
{{% /notice %}}

This is the type of error most prone to misleading debugging when working with DynamoDB GSIs: a role "looks like" it has sufficient permissions to read the table, but still returns `AccessDeniedException` when querying via an index — because it lacks a specific statement for that index's ARN.

#### Two Failure Tiers (Recap from Stream 1)

`ingestion_dlq` (automatic SQS after 3 retries) is separate from `document_processor_fn_dlq` (code proactively writes with tracebacks on unexpected runtime errors) — distinguishing "bad documents" from "code bugs" without needing to read logs to guess (details on [5.3.1](../../5.3-Data-Ingestion/5.3.1-Infrastructure-S3-SQS/) and [5.3.5](../../5.3-Data-Ingestion/5.3.5-Resume-OCR-Error-Handling/)).

#### Intentional Fail-Open with Loud Logging

3 points in the code **intentionally allow errors to pass through** rather than hard blocking — this is a deliberate design decision, not a bug:

| Location | Error Behavior | Rationale |
| --- | --- | --- |
| **Guardrail** (`handler.py:180`, `chat_engine`) | Passes questions/answers through if the moderation API itself fails | "Better to be briefly permissive than bring down the whole app over an auxiliary API blip" |
| **Semantic cache** | Errors always resolve to a cache miss | As stated on [page 5.8.2](../5.8.2-Cache-and-Observability/) — never fail requests due to auxiliary reasons |
| **Cleaning old chunks during re-indexing** (`document_processor/handler.py:296`) | Errors in this step only log a warning and continue | Better to ingest with potential duplicates than fail the entire document over a cleanup step |

```python
def _apply_guardrail(text, source):
    """Run Bedrock Guardrails. Returns (allowed, possibly_masked_text).

    A guardrail failure is treated as fail-open on the text but is logged
    loudly: blocking every request because the moderation API had a blip
    would be a worse outcome than briefly unmoderated traffic for this
    workload. Flip this to fail-closed if your compliance posture requires it.
    """
    try:
        response = bedrock.apply_guardrail(
            guardrailIdentifier=GUARDRAIL_ID,
            guardrailVersion=GUARDRAIL_VERSION,
            source=source,
            content=[{"text": {"text": text}}],
        )
    except Exception:  # noqa: BLE001
        logger.warning("Guardrail evaluation failed (%s); allowing through", source, exc_info=True)
        return True, text

    if response.get("action") == "GUARDRAIL_INTERVENED":
        outputs = response.get("outputs", [])
        message = outputs[0].get("text") if outputs else None
        logger.info("Guardrail intervened on %s", source)
        return False, message or "Yêu cầu này vi phạm chính sách nội dung."

    return True, text
```

{{% notice note %}}
The fail-open decision for Guardrails **is easily reversible** (change 1 line: `raise` instead of log-and-continue) if compliance requirements become stricter in the future. This is a point worth highlighting in reports as a deliberate trade-off, not an oversight.
{{% /notice %}}

#### Bedrock Throttling — Explicit Handling, No Error Swallowing

{{% notice warning %}}
Unlike the 3 fail-open cases above, Bedrock throttling **is not** fail-open — it specifically catches `ThrottlingException`/`TooManyRequestsException`, returns `429 {retryable: true}` so the client knows to wait, and **logs at ERROR level with the exact exception name** so the metric filter on [Stream 3 Monitoring, page 5.5.2](../../5.5-Monitoring/5.5.2-CloudWatch-Alarms/) catches it and feeds into the `bedrock-throttle` alarm.
{{% /notice %}}

```python
from botocore.config import Config

bedrock = boto3.client(
    "bedrock-runtime",
    config=Config(
        retries={"max_attempts": 4, "mode": "adaptive"},
        connect_timeout=5,
        read_timeout=20,
    ),
)
```

The `boto3` client comes pre-configured with `retries={"max_attempts": 4, "mode": "adaptive"}` — automatically backing off when throttled, but **intentionally set low**: API Gateway waits at most **29 seconds**, so excessive retries would only guarantee a timeout instead of returning a useful error to the client.

---

#### Next Content

- [5.8.4 - Backend Testing](../5.8.4-Backend-Testing/)
