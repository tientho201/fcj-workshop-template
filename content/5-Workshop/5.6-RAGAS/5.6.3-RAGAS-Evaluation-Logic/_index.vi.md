---
title: "Logic đánh giá RAG"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.6.3 </b> "
---

{{% notice note %}}
📌 Chính code của `evaluation_runner.py` **tự ghi rõ đây là bản skeleton/placeholder**, không phải production-hardened — dùng để khớp đúng hình dạng với hạ tầng Terraform đã dựng (trang [5.6.1](../5.6.1-EventBridge-Lambda-Container/), [5.6.2](../5.6.2-IAM-Alarm-RAGAS/)) chứ chưa chắc chạy đúng ngay trong mọi trường hợp thực tế.
{{% /notice %}}

#### `_yesterday()` — luôn chạy cho dữ liệu hôm qua

```python
import datetime

def _yesterday() -> str:
    return (datetime.date.today() - datetime.timedelta(days=1)).isoformat()

```

Job chạy theo **batch ngày** (không phải realtime) — mỗi lần chạy (kích hoạt bởi EventBridge Scheduler lúc 2h sáng) sẽ đánh giá toàn bộ dữ liệu hỏi-đáp của **ngày hôm trước**, đảm bảo dữ liệu đã đầy đủ, ổn định khi đánh giá.

#### `_fetch_yesterdays_records` — Scan feedback, Query GSI để ghép đầy đủ

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
📌 **Đã khép kín gap từng ghi nhận trước đây** (xem lại lịch sử ở [trang 5.4.7](../../5.4-Realtime-QA/5.4.7-Alternative-Route/)): hàm `Scan` bảng `feedback` lọc theo ngày như cũ, nhưng **với mỗi dòng feedback tìm được, gọi thêm `_fetch_turn_by_message_id`** (`Query` trên GSI `message_id-index` — đã tạo ở [trang 5.6.2](../5.6.2-IAM-Alarm-RAGAS/)) để lấy lại `question`/`answer`/`retrieved_context` từ `chat_history`, ghép thành **1 record đầy đủ**. `feedback` chỉ đóng vai trò "danh sách cần đánh giá" (lọc theo ngày + có rating), còn dữ liệu thật để chấm RAGAS vẫn lấy từ `chat_history` qua GSI.
{{% /notice %}}

{{% notice warning %}}
**Hành vi cần lưu ý:** dòng feedback nào **không tìm được lượt hỏi tương ứng** qua GSI (turn = `None`) thì bị **bỏ qua âm thầm** (`continue`), không đưa vào tập đánh giá — không raise lỗi, không log riêng. Điều này có thể xảy ra vì **lệch TTL giữa 2 bảng**: `feedback` không có TTL (tồn tại vĩnh viễn), còn `chat_history` có TTL mặc định **30 ngày**. Nếu 1 dòng feedback "sống sót" lâu hơn 30 ngày mà job đánh giá chưa kịp chạy tới, lượt hỏi gốc trong `chat_history` đã bị TTL xóa — dòng feedback đó vĩnh viễn không nối lại được nữa. Job phải chạy đều đặn hàng ngày (đúng như thiết kế lịch của [5.6.1](../5.6.1-EventBridge-Lambda-Container/)) mới tránh được tình huống này.
{{% /notice %}}

#### `_run_ragas` — chạy 4 metric chuẩn của RAGAS

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
📌 **`contexts` ở đây chính là `retrieved_context`** mà `chat_engine` đã lưu kèm mỗi lượt hỏi-đáp ở Luồng 2 (`handler.py:231` — xem lại [trang 5.4.5](../../5.4-Realtime-QA/5.4.5-Sinh-cau-tra-loi-Luu-lich-su/)). Đây chính là lý do vì sao Luồng 2 phải chủ động lưu lại ngữ cảnh đã truy xuất, chứ không chỉ lưu câu trả lời cuối cùng — nếu thiếu `retrieved_context`, Luồng 4 sẽ không thể tính được `context_precision` và `context_recall`, vì 2 metric này cần biết chính xác model đã "nhìn thấy" đoạn văn bản nào trước khi trả lời.
{{% /notice %}}

Hệ thống dùng **chính Amazon Bedrock làm "giám khảo"** (`BedrockChat` + `BedrockEmbeddings`) để chấm điểm 4 chỉ số:

| Metric              | Ý nghĩa                                                                                |
| ------------------- | -------------------------------------------------------------------------------------- |
| `faithfulness`      | Câu trả lời có bám sát context đã truy xuất không (không bịa)                          |
| `answer_relevancy`  | Câu trả lời có đúng trọng tâm câu hỏi không                                            |
| `context_precision` | Trong các đoạn context truy xuất được, tỷ lệ đoạn thực sự liên quan                    |
| `context_recall`    | Context truy xuất được có bao phủ đủ thông tin cần thiết không (so với `ground_truth`) |

#### `_write_results` và `_publish_metrics`

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

- **`_write_results`** ghi **toàn bộ điểm chi tiết từng bản ghi** (không chỉ trung bình) vào S3, theo đường dẫn `evaluation/<ngày>/results.json` — phục vụ truy vết/phân tích sâu sau này khi cần điều tra vì sao 1 câu trả lời cụ thể bị điểm thấp.
- **`_publish_metrics`** chỉ đẩy **trung bình mỗi metric** lên CloudWatch namespace `RAGEvaluation` — đây chính là nguồn dữ liệu cho widget "RAGAS Evaluation Scores" ở [Dashboard Luồng 3](../../5.5-Monitoring/5.5.3-Dashboard-Custom-Metrics/) và cho alarm `ragas-faithfulness-low` ở [trang 5.6.2](../5.6.2-IAM-Alarm-RAGAS/).

---

#### Nội dung tiếp theo

- [5.6.4 - Kiểm thử và lưu ý triển khai thực tế](../5.6.4-Kiem-thu-Luu-y/)
