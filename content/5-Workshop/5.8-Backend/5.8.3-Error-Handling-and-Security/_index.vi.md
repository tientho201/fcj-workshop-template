---
title: "Xử lý lỗi & Bảo mật"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.8.3 </b> "
---

#### Nguyên tắc IAM xuyên suốt cả 2 Lambda

Không có policy nào dùng `Resource: "*"` **trừ khi API của AWS buộc phải vậy** (Textract, X-Ray, quản lý ENI của VPC, `cloudwatch:PutMetricData`) — và **mỗi ngoại lệ đều có comment giải thích tại sao**, cộng thêm `condition` thu hẹp phạm vi nếu API cho phép (ví dụ `PutMetricData` bị khóa cứng vào đúng namespace `RAGEvaluation` bằng `cloudwatch:namespace` condition — xem lại [Luồng 4, trang 5.6.2](../../5.6-RAGAS/5.6.2-IAM-Alarm-RAGAS/)).

{{% notice tip %}}
**AWS managed policy** (`AWSLambdaBasicExecutionRole`, `AWSLambdaVPCAccessExecutionRole`) **cố tình không dùng** vì chúng đi kèm phạm vi rộng hơn cần thiết (`logs:*` trên **mọi** log group trong account) — thay vào đó mỗi Lambda có policy tự viết, scope đúng **1 log group của chính nó**.
{{% /notice %}}

#### Pattern lặp lại: quyền trên bảng gốc không tự động bao gồm quyền trên GSI

{{% notice warning %}}
📌 **Đây là pattern IAM xuất hiện ở ít nhất 2 nơi trong dự án**, đáng ghi nhận như 1 nguyên tắc chung: `evaluation_runner` cần thêm **1 statement IAM riêng** để `Query` GSI `message_id-index` (`"${chat_history_table_arn}/index/*"`) — quyền `Query`/`Scan`/`GetItem` đã cấp trên chính bảng `chat_history` **không tự động cho phép Query trên index của nó** (chi tiết đầy đủ ở [Luồng 4, trang 5.6.2](../../5.6-RAGAS/5.6.2-IAM-Alarm-RAGAS/)).

**Cùng pattern này lặp lại ở `modules/ingestion/main.tf`** cho GSI `document-id-index` của bảng `child_chunks` — dùng để xóa toàn bộ chunk cũ theo `document_id` khi re-index (xem lại [Luồng 1, trang 5.3.4](../../5.3-Data-Ingestion/5.3.4-Chunking-Embedding/)). `document_processor` cũng cần 1 statement IAM riêng cho GSI này, tách biệt khỏi quyền trên bảng `child_chunks` gốc.
{{% /notice %}}

Đây là kiểu lỗi dễ gây debug nhầm hướng nhất khi làm việc với DynamoDB GSI: role "trông có vẻ" đủ quyền đọc bảng, nhưng vẫn báo `AccessDeniedException` khi Query qua index — vì thiếu đúng 1 statement riêng cho ARN của index đó.

#### Hai tầng thất bại (nhắc lại từ Luồng 1)

`ingestion_dlq` (SQS tự động sau 3 lần retry) tách biệt với `document_processor_fn_dlq` (code chủ động ghi kèm traceback khi gặp lỗi runtime bất ngờ) — phân biệt "tài liệu xấu" với "code lỗi" mà không cần đọc log để đoán (chi tiết ở [5.3.1](../../5.3-Data-Ingestion/5.3.1-Infrastructure-S3-SQS/) và [5.3.5](../../5.3-Data-Ingestion/5.3.5-Resume-OCR-Error-Handling/)).

#### Fail-open có chủ đích, ghi log to

3 điểm trong code **cố tình cho lỗi đi qua** thay vì chặn cứng — đây là quyết định thiết kế có cân nhắc, không phải bug:

| Vị trí                                                              | Hành vi khi lỗi                                                   | Lý do                                                                                                  |
| ------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Guardrail** (`handler.py:180`, `chat_engine`)                     | Cho câu hỏi/câu trả lời đi qua nếu bản thân API kiểm duyệt bị lỗi | "Thà thoáng một lúc còn hơn sập cả app vì một API phụ trợ trục trặc"                                   |
| **Semantic cache**                                                  | Lỗi luôn quy về cache miss                                        | Đã nêu ở [trang 5.8.2](../5.8.2-Cache-va-Observability/) — không bao giờ fail request vì lý do phụ trợ |
| **Dọn chunk cũ khi re-index** (`document_processor/handler.py:296`) | Lỗi bước này chỉ log cảnh báo và tiếp tục                         | Thà ingest với khả năng trùng lặp còn hơn fail cả tài liệu vì một bước dọn dẹp                         |

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
Quyết định fail-open ở Guardrail **có thể đảo ngược dễ dàng** (đổi 1 dòng: `raise` thay vì log-and-continue) nếu yêu cầu compliance khắt khe hơn trong tương lai. Đây là điểm đáng nêu trong báo cáo như một trade-off có cân nhắc, không phải thiếu sót.
{{% /notice %}}

#### Bedrock throttling — xử lý rõ ràng, không nuốt lỗi

{{% notice warning %}}
Khác với 3 trường hợp fail-open ở trên, throttle từ Bedrock **không** fail-open — bắt riêng `ThrottlingException`/`TooManyRequestsException`, trả `429 {retryable: true}` cho client biết nên chờ, đồng thời **log ở mức ERROR với nguyên tên exception** để metric filter phía [Luồng 3 Monitoring, trang 5.5.2](../../5.5-Monitorning/5.5.2-CloudWatch-Alarms/) bắt được và gộp vào alarm `bedrock-throttle`.
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

`boto3` client cấu hình sẵn `retries={"max_attempts": 4, "mode": "adaptive"}` — tự lùi lại (backoff) khi bị throttle, nhưng **giới hạn thấp có chủ đích**: API Gateway chỉ chờ tối đa **29 giây**, retry quá đà chỉ đảm bảo timeout thay vì trả lỗi có ích cho client.

---

#### Nội dung tiếp theo

- [5.8.4 - Kiểm thử Backend](../5.8.4-Backend-Testing/)
