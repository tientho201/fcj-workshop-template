---
title: "RAGAS Evaluation Logic"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.6.3 </b> "
---

{{% notice note %}}
📌 The code in `evaluation_runner.py` **explicitly states that it is a skeleton/placeholder**, not a production-hardened implementation — designed to match the exact shape of the provisioned Terraform infrastructure (pages [5.6.1](../5.6.1-EventBridge-Lambda-Container/), [5.6.2](../5.6.2-IAM-Alarm-RAGAS/)), though it may require further hardening for real-world production cases.
{{% /notice %}}

#### `_yesterday()` — always runs for yesterday's data

```python
import datetime

def _yesterday() -> str:
    return (datetime.date.today() - datetime.timedelta(days=1)).isoformat()
```

The job runs in **daily batches** (not real-time) — each run (triggered by EventBridge Scheduler at 2:00 AM) evaluates all Q&A data from **the previous day**, ensuring that data is complete and stable at evaluation time.

#### `_fetch_yesterdays_records` — Scans feedback, Queries GSI to join full turn data

```python
def _fetch_turn_by_message_id(chat_history_table, message_id: str) -> dict | None:
    response = chat_history_table.query(
        IndexName=CHAT_HISTORY_MESSAGE_ID_INDEX,
        KeyConditionExpression=Key("message_id").eq(message_id),
        Limit=1,
    )
    items = response.get("Items", [])
    return items[0] if items else None

def _fetch_yesterdays_records(date_str: str) -> list[dict]:
    feedback_table = dynamodb.Table(FEEDBACK_TABLE)
    chat_history_table = dynamodb.Table(CHAT_HISTORY_TABLE)

    feedback_records = []
    scan_kwargs = {"FilterExpression": "begins_with(created_at, :d)", "ExpressionAttributeValues": {
        ":d": date_str}}
    while True:
        resp = feedback_table.scan(**scan_kwargs)
        feedback_records.extend(resp.get("Items", []))
        if "LastEvaluatedKey" not in resp:
            break
        scan_kwargs["ExclusiveStartKey"] = resp["LastEvaluatedKey"]

    records = []
    for feedback in feedback_records:
        message_id = feedback.get("message_id")
        turn = _fetch_turn_by_message_id(chat_history_table, message_id) if message_id else None
        if turn is None:
            continue
        records.append(
            {
                **feedback,
                "question": turn.get("question", ""),
                "answer": turn.get("answer", ""),
                "retrieved_context": turn.get("retrieved_context", []),
            }
        )
    return records
```

{{% notice note %}}
📌 **Resolved the previously documented gap** (refer back to history on [page 5.4.7](../../5.4-Realtime-QA/5.4.7-Alternative-Route/)): the `Scan` function on the `feedback` table filters by date as before, but **for each found feedback row, it calls `_fetch_turn_by_message_id`** (a `Query` on the `message_id-index` GSI — created on [page 5.6.2](../5.6.2-IAM-Alarm-RAGAS/)) to retrieve `question`/`answer`/`retrieved_context` from `chat_history`, constructing **1 complete record**. The `feedback` table only serves as the "evaluation list" (filtered by date + containing ratings), while the actual data for RAGAS scoring is fetched from `chat_history` via GSI.
{{% /notice %}}

{{% notice warning %}}
**Behavior to note:** Any feedback row that **cannot find its corresponding turn** via GSI (turn = `None`) is **silently skipped** (`continue`) and excluded from the evaluation set — no error is raised and no separate log is generated. This can happen due to **TTL mismatch between the two tables**: `feedback` has no TTL (persists indefinitely), while `chat_history` has a default TTL of **30 days**. If a feedback row "survives" longer than 30 days before the evaluation job processes it, the original turn in `chat_history` will have been deleted by TTL — rendering that feedback row permanently unjoinable. The job must run regularly every day (as scheduled in [5.6.1](../5.6.1-EventBridge-Lambda-Container/)) to prevent this situation.
{{% /notice %}}

#### `_run_ragas` — runs the 4 standard RAGAS metrics

```python
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall
from ragas.llms import BedrockChat
from ragas.embeddings import BedrockEmbeddings

def _run_ragas(records: list[dict]) -> dict:
    """Score records with RAGAS. Returns per-metric averages + the raw
    per-record scores (the latter go to S3, the former to CloudWatch).
    """
    from datasets import Dataset
    from langchain_aws import BedrockChat, BedrockEmbeddings
    from ragas import evaluate
    from ragas.metrics import answer_relevancy, context_precision, context_recall, faithfulness

    if not records:
        return {"averages": {}, "per_record": []}

    dataset = Dataset.from_list(
        [
            {
                "question": r.get("question", ""),
                "answer": r.get("answer", ""),
                "contexts": r.get("retrieved_context", []),
                "ground_truth": r.get("ground_truth", ""),
            }
            for r in records
        ]
    )

    judge_llm = BedrockChat(model_id=RAGAS_JUDGE_MODEL_ID)
    judge_embeddings = BedrockEmbeddings()

    result = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy,
                 context_precision, context_recall],
        llm=judge_llm,
        embeddings=judge_embeddings,
    )

    df = result.to_pandas()
    averages = {
        "faithfulness": float(df["faithfulness"].mean()),
        "answer_relevancy": float(df["answer_relevancy"].mean()),
        "context_precision": float(df["context_precision"].mean()),
        "context_recall": float(df["context_recall"].mean()),
    }
    return {"averages": averages, "per_record": df.to_dict(orient="records")}
```

{{% notice note %}}
📌 **`contexts` here refers directly to `retrieved_context`** saved by `chat_engine` alongside each Q&A turn in Stream 2 (`handler.py:231` — refer back to [page 5.4.5](../../5.4-Realtime-QA/5.4.5-Answer-Generation-History-Storage/)). This is precisely why Stream 2 proactively stores retrieved contexts rather than just the final answer — without `retrieved_context`, Stream 4 would be unable to compute `context_precision` and `context_recall`, as both metrics require knowing exactly which text chunks the model "saw" before answering.
{{% /notice %}}

The system uses **Amazon Bedrock itself as the "judge"** (`BedrockChat` + `BedrockEmbeddings`) to evaluate 4 metrics:

| Metric              | Meaning                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------- |
| `faithfulness`      | Whether the answer strictly adheres to the retrieved context (no hallucination)             |
| `answer_relevancy`  | Whether the answer directly addresses the core user question                                |
| `context_precision` | Among all retrieved context chunks, the proportion of truly relevant ones                   |
| `context_recall`    | Whether the retrieved context covers all necessary information (compared to `ground_truth`) |

#### `_write_results` and `_publish_metrics`

```python
def _write_results(date_str: str, results: dict) -> None:
    s3.put_object(
        Bucket=RESULTS_BUCKET,
        Key=f"evaluation/{date_str}/results.json",
        Body=json.dumps(results).encode("utf-8"),
        ContentType="application/json",
    )

def _publish_metrics(averages: dict) -> None:
    if not averages:
        return
    cloudwatch.put_metric_data(
        Namespace="RAGEvaluation",
        MetricData=[{"MetricName": name.capitalize(), "Value": value, "Unit": "None"}
                    for name, value in averages.items()],
    )
```

- **`_write_results`** writes **detailed per-record scores** (not just averages) to S3 at `evaluation/<date>/results.json` — enabling deep-dive tracing and analysis later when investigating why a specific answer received a low score.
- **`_publish_metrics`** pushes only **metric averages** to the CloudWatch namespace `RAGEvaluation` — serving as the data source for the "RAGAS Evaluation Scores" widget on the [Stream 3 Dashboard](../../5.5-Monitoring/5.5.3-Dashboard-Custom-Metrics/) and triggering the `ragas-faithfulness-low` alarm on [page 5.6.2](../5.6.2-IAM-Alarm-RAGAS/).

---

#### Next Content

- [5.6.4 - Testing and Practical Deployment Notes](../5.6.4-Testing-Practical-Notes/)
