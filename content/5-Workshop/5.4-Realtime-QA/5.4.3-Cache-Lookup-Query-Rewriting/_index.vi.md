---
title: "Truy xuất Cache và Rewrite Query"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.4.3 </b> "
---

Một Lambda `chat_engine` duy nhất phục vụ cả 4 route đã khai báo ở trang [5.4.1](../5.4.1-API-Gateway-Cognito/), rẽ nhánh theo `resource_path`/`http_method` ngay đầu `lambda_handler` (`handler.py:413`):

```python
def lambda_handler(event, context):
    # One Lambda serves the whole small API surface (chat, upload, status,
    # OCR decision, feedback) rather than one function per route. They
    # share the same config, IAM shape and deployment unit, so splitting
    # them would mean more log groups/roles/packages to keep in step for no
    # isolation benefit.
    resource_path = event.get("resource") or event.get("path") or ""
    http_method = event.get("httpMethod", "POST")

    if resource_path.endswith("/documents") and http_method == "POST":
        try:
            return _handle_upload(event)
        except Exception:  # noqa: BLE001
            logger.exception("Upload failed")
            return _response(500, {"error": "Không tải được tài liệu lên."})

    if resource_path.endswith("/documents-decision") and http_method == "POST":
        try:
            return _handle_ocr_decision(event)
        except Exception:  # noqa: BLE001
            logger.exception("OCR decision failed")
            return _response(500, {"error": "Không xử lý được quyết định OCR."})

    if resource_path.endswith("/feedback") and http_method == "POST":
        try:
            return _handle_feedback(event)
        except Exception:  # noqa: BLE001
            logger.exception("Feedback submission failed")
            return _response(500, {"error": "Không ghi được phản hồi."})

    if resource_path.endswith("/status"):
        try:
            return _handle_status(event)
        except Exception:  # noqa: BLE001
            logger.exception("Status lookup failed")
            return _response(500, {"error": "Không đọc được trạng thái ingestion."})

    return _handle_chat(event)
```

Trang này và 2 trang tiếp theo trình bày chi tiết route chính `_handle_chat` — chạy đúng theo thứ tự ghi trong docstring đầu file. Hai bước đầu tiên: **cache lookup** và **query rewriting**.

#### Bước 1 — Semantic cache lookup

```python
def _handle_chat(event):
    semantic_cache = SemanticCache()
    tracer = Tracer()

    try:
        body = json.loads(event.get("body") or "{}")
        question = (body.get("question") or "").strip()
        session_id = body.get("session_id") or str(uuid.uuid4())
        message_id = str(uuid.uuid4())

        if not question:
            return _response(400, {"error": "Field 'question' is required."})

        history = _recent_history(session_id)

        cacheable = not history

        if cacheable:
            cached_answer = semantic_cache.get(question)
            if cached_answer:
                _emit_metric("CacheHit")
                logger.info("Semantic cache hit")
                tracer.step(
                    "cache",
                    "Semantic cache",
                    detail="CACHE HIT — bỏ qua toàn bộ retrieval và generation",
                )
                return _response(
                    200,
                    {
                        "answer": cached_answer,
                        "session_id": session_id,
                        "message_id": message_id,
                        "cached": True,
                        "sources": [],
                        "trace": tracer.to_dict(),
                    },
                )
            _emit_metric("CacheMiss")
            tracer.step("cache", "Semantic cache", detail="CACHE MISS — chạy pipeline đầy đủ")
        else:
            tracer.skip(
                "cache",
                "Semantic cache",
                detail=f"Bỏ qua — phiên đã có {len(history)} lượt, câu hỏi có thể phụ thuộc ngữ cảnh",
            )
```

{{% notice info %}}
**Vì sao chỉ cache khi chưa có lịch sử?** Cache khóa theo câu hỏi thô (`hash_question`). Nếu giữa chừng hội thoại người dùng hỏi tiếp kiểu phụ thuộc ngữ cảnh (ví dụ _"còn cái thứ hai thì sao?"_) mà vẫn dùng cache, hệ thống có thể trả nhầm câu trả lời của một phiên hội thoại khác đã hỏi câu tương tự trước đó nhưng trong ngữ cảnh hoàn toàn khác. Giới hạn cache chỉ ở câu hỏi đầu tiên của phiên là cách đơn giản và an toàn để tránh rủi ro này.
{{% /notice %}}

{{% notice tip %}}
**Cơ chế xử lý Cache Hit và Cache Miss:**

- **Khi CACHE HIT** (Tìm thấy câu trả lời trong Redis):
  - Phát metric `CacheHit` lên CloudWatch để theo dõi hiệu năng.
  - Trả kết quả ngay lập tức (`200 OK`) cho client kèm theo nội dung câu trả lời đã lưu trong Redis.
  - **Bỏ qua toàn bộ các bước phía sau**: Không cần tốn chi phí và thời gian tìm kiếm vector trong DynamoDB hay gọi Claude 3/Bedrock sinh câu trả lời, giúp phản hồi cực nhanh (vài chục ms).
- **Khi CACHE MISS** (hoặc Bỏ qua Cache do phiên đã có lịch sử):
  - Ghi log/trace bước này và tiếp tục chuyển câu hỏi sang pipeline RAG đầy đủ (`Query Rewriting` ➔ `Hybrid Search` ➔ `Bedrock Generation`).
    {{% /notice %}}

#### Bước 2 — Query Rewriting (Claude Haiku)

```python
def _rewrite_query(question, history):
    """Rewrite a follow-up into a standalone question.

    Skipped entirely when there's no history — a first turn is standalone by
    definition, and skipping saves a model call on the most common path.
    Falls back to the original question if the rewrite fails for any reason;
    a degraded query beats a failed request.
    """
    if not history:
        return question

    transcript = "\n".join(
        f"User: {turn.get('question', '')}\nAssistant: {turn.get('answer', '')}"
        for turn in history
    )
    prompt = (
        "Rewrite the user's latest question so it stands alone without the "
        "conversation history — resolve pronouns and implicit references. "
        "Keep the original language. Output ONLY the rewritten question, "
        "nothing else.\n\n"
        f"<conversation_history>\n{transcript}\n</conversation_history>\n\n"
        f"<latest_question>\n{question}\n</latest_question>"
    )

    try:
        rewritten = _invoke_claude(
            HAIKU_MODEL_ID, prompt, max_tokens=200, temperature=0.0
        ).strip()
        if rewritten:
            logger.info("Rewrote query: %r -> %r", question, rewritten)
            return rewritten
    except Exception:
        logger.warning("Query rewriting failed; using the original question", exc_info=True)

    return question
```

Bước này **chỉ chạy khi phiên đã có lịch sử hội thoại**, với mục đích biến câu hỏi phụ thuộc ngữ cảnh (ví dụ _"còn cái thứ hai thì sao?"_) thành câu hỏi độc lập, đầy đủ ngữ nghĩa để bước retrieval ở trang tiếp theo có thể match đúng — vì retrieval chỉ nhìn thấy câu hỏi, không có ngữ cảnh hội thoại trước đó.

{{% notice tip %}}
Dùng **Claude Haiku** (model nhỏ, nhanh, rẻ) riêng cho tác vụ rewrite thay vì dùng chung model chính — đây là lý do IAM Role ở trang [5.4.2](../5.4.2-Cache-Guardrails-IAM/) scope đúng 3 model ID thay vì 2 (embedding + main model). Nếu bước rewrite gặp lỗi (timeout, model lỗi...), code **fallback về câu hỏi gốc thay vì raise exception** — đảm bảo một lỗi ở bước phụ trợ này không làm fail toàn bộ request của người dùng.
{{% /notice %}}

---

#### Nội dung tiếp theo

- [5.4.4 - Hybrid Search và Retrieval](../5.4.4-Hybrid-Search-Retrieval/)
