---
title: "Answer Generation and History Storage"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.4.5 </b> "
---

After obtaining context from page [5.4.4](../5.4.4-Hybrid-Search-Retrieval/), the final 3 steps of `_handle_chat` are answer generation, output moderation, and storing data for caching, chat history, and subsequent quality evaluation.

#### Step 6 — Answer Generation using Claude

```python
ANSWER_SYSTEM_PROMPT = (
    "You are a retrieval-augmented assistant. Answer using ONLY the provided "
    "context. If the context does not contain the answer, say plainly that you "
    "don't have enough information — never invent facts or fill gaps from prior "
    "knowledge. Cite the source document id(s) you used. Reply in the same "
    "language as the user's question."
)
def _invoke_claude(model_id, prompt, system=None, max_tokens=MAX_ANSWER_TOKENS, temperature=0.2):
    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_tokens,
        "temperature": temperature,
        "messages": [{"role": "user", "content": prompt}],
    }
    if system:
        body["system"] = system

    response = bedrock.invoke_model(modelId=model_id, body=json.dumps(body))
    payload = json.loads(response["body"].read())
    return "".join(block.get("text", "") for block in payload.get("content", []))

def _build_answer_prompt(question, parents):
    context_blocks = "\n\n".join(
        f"<document id=\"{parent.get('document_id', 'unknown')}\">\n"
        f"{parent.get('parent_text', '')}\n</document>"
        for parent in parents
    )
    return (
        f"<context>\n{context_blocks}\n</context>\n\n"
        f"<question>\n{question}\n</question>\n\n"
        "Answer the question using only the context above."
    )

answer = _invoke_claude(
    TEXT_MODEL_ID,
    _build_answer_prompt(search_query, parents),
    system=ANSWER_SYSTEM_PROMPT,
)
```

- Prompt design follows `Anthropic Claude Prompt Engineering` standards:
  - Separate System Prompt (`ANSWER_SYSTEM_PROMPT`): Passed directly into the system parameter of the Anthropic API in English for maximum compliance. Forces the model to use ONLY the context provided inside the `<context>` tag, strictly preventing hallucination outside the data, while automatically responding in the same language as the user's question.
  - Wrap context in `XML tags` (`<document id="...">`): Helps Claude clearly identify boundaries between documents and cite source `document_id`s accurately.
  - Configured `temperature = 0.2`: Minimizes random creativity, forcing the model to strictly adhere to factual context.
  - Integration benefits: Enables users to verify which document an answer originates from, while standardizing output data for Stream 4 (RAGAS) to measure `faithfulness` and `context_precision` later.

#### Step 7 — Caching and History Persistence `handler.py:601-621`

```python
    if cacheable:
        semantic_cache.set(question, answer)

    _persist_turn(session_id, question, answer, parents)
    tracer.step(
        "persist",
        "Ghi cache + lịch sử",
        detail="Đã lưu vào DynamoDB" + (" và semantic cache" if cacheable else ""),
    )

    return _response(
        200,
        {
            "answer": answer,
            "session_id": session_id,
            "message_id": message_id,
            "cached": False,
            "rewritten_query": search_query if search_query != question else None,
            "sources": sorted({p.get("document_id", "") for p in parents}),
            "trace": tracer.to_dict(),
        },
    )
```

Cache Writing (`semantic_cache.set`): If `cacheable = True` (the first question turn of a session without prior history, as described on page [5.4.3](../5.4.3-Cache-Lookup-Query-Rewriting/)), the question-answer pair is saved to ElastiCache Redis.
Persisting Conversation History (`_persist_turn`): Writes the chat turn data to the `chat_history` DynamoDB table, including:

- `session_id`, `timestamp`, `question`, `answer`.
- `retrieved_context`: Array containing text content (`parent_text`) of parent chunks used as context.
- `source`: Array of source document IDs (`document_id`).
- `expires_at`: Automatic row deletion expiration (TTL) in days configured by `CHAT_HISTORY_TTL_DAYS`.
Client Response (`_response`): Returns HTTP Status 200 containing the answer, `session_id`, `message_id`, sorted `sources` list, `rewritten_query` (if applicable), and a `trace` array detailing timing/status per step.

{{% notice note %}}
The `retrieved_context` and `source` fields stored alongside in `chat_history` are not just for displaying chat history — this is the exact data that Stream 4 (RAGAS Evaluation) reads to calculate `context_precision` and `context_recall` metrics during daily automated RAG quality evaluations. Pre-storing these two fields eliminates the need for Stream 4 to rerun costly retrieval steps during evaluation.
{{% /notice %}}

---

#### Next Content

- [5.4.6 - Error Handling and OCR Decision Integration](../5.4.6-Error-Handling-OCR-Decision/)
