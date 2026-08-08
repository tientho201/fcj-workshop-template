---
title: "Cache Lookup and Query Rewriting"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.4.3 </b> "
---

A single `chat_engine` Lambda serves all 4 routes declared on page [5.4.1](../5.4.1-API-Gateway-Cognito/), branching based on `resource_path`/`http_method` right at the start of `lambda_handler` (`handler.py:413`):

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

This page and the next two pages detail the main `_handle_chat` route — running in the exact sequence specified in the top file docstring. The first two steps are: **cache lookup** and **query rewriting**.

#### Step 1 — Semantic cache lookup

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
**Why only cache when there is no history?** The cache keys on the raw question (`hash_question`). If mid-conversation the user asks a context-dependent follow-up (e.g., _"what about the second one?"_) and cache is still used, the system might return a wrong answer from another session that asked a similar question in a completely different context. Restricting caching to the first question in a session is a simple and safe way to avoid this risk.
{{% /notice %}}

{{% notice tip %}}
**Cache Hit and Cache Miss Handling Mechanism:**

- **On CACHE HIT** (Answer found in Redis):
  - Emits a `CacheHit` metric to CloudWatch for performance monitoring.
  - Returns the result immediately (`200 OK`) to the client along with the answer stored in Redis.
  - **Skips all subsequent steps**: Eliminates costs and latency of vector search in DynamoDB or calling Claude 3/Bedrock for answer generation, enabling ultra-fast responses (a few dozen ms).
- **On CACHE MISS** (or Cache Bypassed due to existing session history):
  - Logs/traces this step and forwards the question to the full RAG pipeline (`Query Rewriting` ➔ `Hybrid Search` ➔ `Bedrock Generation`).
    {{% /notice %}}

#### Step 2 — Query Rewriting (Claude Haiku)

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

This step **only runs when the session has conversation history**, aiming to transform context-dependent questions (e.g., _"what about the second one?"_) into standalone, semantically complete queries so that the subsequent retrieval step can match correctly — since retrieval only sees the question without prior conversation context.

{{% notice tip %}}
Using **Claude Haiku** (a small, fast, cheap model) specifically for the rewrite task instead of sharing the main model — this is why the IAM Role on page [5.4.2](../5.4.2-Cache-Guardrails-IAM/) scopes exactly 3 model IDs instead of 2 (embedding + main model). If the rewrite step encounters an error (timeout, model error...), the code **falls back to the original question instead of raising an exception** — ensuring an issue in this auxiliary step does not fail the user's entire request.
{{% /notice %}}

---

#### Next Content

- [5.4.4 - Hybrid Search and Retrieval](../5.4.4-Hybrid-Search-Retrieval/)
