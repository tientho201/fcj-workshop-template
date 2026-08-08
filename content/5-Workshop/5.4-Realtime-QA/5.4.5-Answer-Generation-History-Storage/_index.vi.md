---
title: "Sinh câu trả lời và lưu lịch sử"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.4.5 </b> "
---

Sau khi có ngữ cảnh từ trang [5.4.4](../5.4.4-Hybrid-Search-Retrieval/), 3 bước cuối của `_handle_chat` là sinh câu trả lời, kiểm duyệt đầu ra, và lưu lại dữ liệu phục vụ cache/lịch sử/đánh giá chất lượng sau này.

#### Bước 6 — Sinh câu trả lời bằng Claude

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

- Prompt được thiết kế theo chuẩn `Anthropic Claude Prompt Engineering`:
  - Phân tách System Prompt riêng biệt (`ANSWER_SYSTEM_PROMPT`): Đặt trực tiếp vào tham số system của Anthropic API bằng tiếng Anh để đạt hiệu quả tuân thủ cao nhất. Ép model chỉ dùng đúng ngữ cảnh được cung cấp trong thẻ `<context>`, tuyệt đối không bịa đặt thông tin ngoài dữ liệu (`hallucination`), đồng thời tự động phản hồi theo cùng ngôn ngữ của câu hỏi.
  - Bọc ngữ cảnh bằng `thẻ XML` (`<document id="...">`): Giúp Claude nhận diện rõ ranh giới từng văn bản và trích dẫn chuẩn xác document_id nguồn.
  - Cấu hình `temperature = 0.2`: Hạn chế tối đa tính sáng tạo ngẫu nhiên, buộc model bám sát thông tin thực tế.
  - Ý nghĩa tích hợp: Giúp người dùng kiểm chứng được câu trả lời đến từ tài liệu nào, đồng thời chuẩn hóa dữ liệu đầu ra để Luồng 4 (RAGAS) đo lường độ chính xác (`faithfulness`) và độ liên quan (`context_precision`) sau này.

#### Bước 7 — Ghi cache và lưu lịch sử `handler.py:601-621`

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

Ghi Cache (semantic_cache.set): Nếu `cacheable = True` (lượt câu hỏi đầu tiên của phiên chưa có lịch sử, như đã nói ở trang [5.4.3](../5.4.3-Cache-Lookup-Query-Rewriting/)), cặp câu hỏi - câu trả lời sẽ được lưu vào ElastiCache Redis.
Lưu lịch sử hội thoại (\_persist_turn): Ghi dữ liệu lượt chat vào bảng DynamoDB chat_history bao gồm:

- session_id, timestamp, question, answer.
- retrieved_context: Mảng chứa nội dung (parent_text) các chunk cha đã dùng làm ngữ cảnh.
- source: Mảng các ID tài liệu nguồn (document_id).
- expires_at: Hạn tự động xóa dòng (TTL) tính theo số ngày cấu hình (CHAT_HISTORY_TTL_DAYS).
  Phản hồi Client (\_response): Trả về HTTP Status 200 chứa câu trả lời, session_id, danh sách sources đã sắp xếp, truy vấn đã rewrite (rewritten_query nếu có), và mảng trace thống kê chi tiết thời gian/trạng thái từng bước.

{{% notice note %}}
Hai trường retrieved_context và source được lưu kèm trong chat_history không chỉ để hiển thị lại lịch sử hội thoại — đây chính là dữ liệu mà Luồng 4 (RAGAS Evaluation) sẽ đọc để tính các chỉ số context_precision và context_recall khi đánh giá tự động chất lượng RAG hàng ngày. Thiết kế lưu sẵn 2 trường này giúp Luồng 4 không cần chạy lại bước retrieval tốn kém khi đánh giá.
{{% /notice %}}

---

#### Nội dung tiếp theo

- [5.4.6 - Xử lý lỗi và kết nối OCR Decision](../5.4.6-Error-Handling-OCR-Decision/)
