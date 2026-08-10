---
title: "Parent-Child Chunking và Embedding"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.3.4 </b> "
---

Sau khi có văn bản đã trích xuất (từ trang [5.3.3](../5.3.3-Text-Extraction/)), bước tiếp theo là chia nhỏ tài liệu, sinh vector embedding, và lưu vào 2 bảng DynamoDB đã tạo ở trang [5.3.2](../5.3.2-Infrastructure-DynamoDB-IAM/).

#### Parent-Child Chunking

```python
def chunk_document(text, document_id):
    normalised = text.replace("\r\n", "\n").strip()
    if not normalised:
        return [], []

    parents = []
    children = []
    child_counter = 0

    for parent_index, parent_text in enumerate(_split_on_boundary(normalised, PARENT_CHARS)):
        parent_id = f"{document_id}#{parent_index}#{uuid.uuid4().hex[:8]}"
        parents.append(
            {
                "parent_id": parent_id,
                "document_id": document_id,
                "parent_text": parent_text,
                "parent_index": parent_index,
            }
        )

        for child_text in _split_children(parent_text):
            children.append(
                {
                    "parent_id": parent_id,
                    "document_id": document_id,
                    "child_text": child_text,
                    "chunk_index": child_counter,
                }
            )
            child_counter += 1

    return parents, children
```

Văn bản được tách thành 2 loại chunk song song:

- **Parent chunk** (~1000-1500 token) — chỉ để làm ngữ cảnh cho LLM khi sinh câu trả lời, không đem đi embed.
- **Child chunk** (~200-300 token) — sẽ được embed và dùng để tìm kiếm chính xác hơn.

Trước khi ghi dữ liệu mới, Lambda **xóa toàn bộ chunk cũ của cùng `document_id`** (nếu người dùng re-upload cùng key) để tránh trùng lặp dữ liệu khi ingest lại.

```python
def _delete_old_chunks(document_id):
    removed = 0
    query_kwargs = {
        "IndexName": "document-id-index",
        "KeyConditionExpression": Key("document_id").eq(document_id),
        "ProjectionExpression": "chunk_id",
    }
    with child_table.batch_writer() as batch:
        while True:
            response = child_table.query(**query_kwargs)
            for item in response.get("Items", []):
                batch.delete_item(Key={"chunk_id": item["chunk_id"]})
                removed += 1
            if "LastEvaluatedKey" not in response:
                break
            query_kwargs["ExclusiveStartKey"] = response["LastEvaluatedKey"]
    return removed
```

#### Lưu Parent Chunks vào DynamoDB

```python
def _store_parents(parents):
    with parent_table.batch_writer() as batch:
        for parent in parents:
            batch.put_item(Item=parent)
```

Chỉ lưu text thô — phục vụ tra cứu ngữ cảnh rộng bằng `parent_id` lúc trả lời câu hỏi ở Luồng 2.

#### Embed Child Chunks qua Amazon Bedrock

```python
def embed_texts(model_id, texts, input_type):
    if not texts:
        return []

    if _is_cohere(model_id):
        vectors = []
        for start in range(0, len(texts), _COHERE_BATCH_LIMIT):
            batch = texts[start : start + _COHERE_BATCH_LIMIT]
            response = _bedrock.invoke_model(
                modelId=model_id,
                body=json.dumps(
                    {
                        "texts": batch,
                        "input_type": input_type,
                        "truncate": "END",
                    }
                ),
            )
            vectors.extend(json.loads(response["body"].read())["embeddings"])
        return vectors

    # Titan-style: one call per text.
    vectors = []
    for text in texts:
        response = _bedrock.invoke_model(
            modelId=model_id,
            body=json.dumps({"inputText": text}),
        )
        vectors.append(json.loads(response["body"].read())["embedding"])
    return vectors
```

Model dùng để embedding được lấy từ biến môi trường `BEDROCK_EMBEDDING_MODEL_ID`, khớp đúng với ARN đã scope trong IAM Role ở trang 5.3.2 — đảm bảo Lambda chỉ gọi được đúng 1 model, không thể gọi các model Bedrock khác.

#### Lưu Child Chunks: vector nhị phân + BM25 term frequency

```python
def _index_children(children, vectors):
    if len(vectors) != len(children):
        raise RuntimeError(
            f"Embedding count mismatch: got {len(vectors)} vectors for {len(children)} chunks"
        )

    with child_table.batch_writer() as batch:
        for child, vector in zip(children, vectors):
            tokens = bm25.tokenize(child["child_text"])
            batch.put_item(
                Item={
                    "chunk_id": f"{child['document_id']}#{child['chunk_index']}",
                    "child_text": child["child_text"],
                    # Packed + L2-normalized — see vector_store.py. Stored as
                    # a Binary attribute, not a List<Number>: far cheaper to
                    # write and, more importantly, far cheaper for
                    # chat-engine to deserialize on every query-time scan.
                    "embedding": vector_store.pack_vector(vector),
                    # BM25 term stats for the lexical half of hybrid search
                    # (see bm25.py). Stored as a JSON string rather than a
                    # native DynamoDB Map<String,Number>: one String
                    # attribute is cheaper to write/read than N separate
                    # map-entry attribute values, same reasoning as the
                    # packed embedding above.
                    "term_freq_json": json.dumps(bm25.term_frequencies(tokens)),
                    "doc_length": len(tokens),
                    "parent_id": child["parent_id"],
                    "document_id": child["document_id"],
                    "chunk_index": child["chunk_index"],
                }
            )
```

{{% notice note %}}
`vector_store.pack_vector()` đóng gói vector thành **Binary attribute** (`embedding`) thay vì `List<Number>` — tiết kiệm đáng kể chi phí lưu trữ và băng thông đọc/ghi trên DynamoDB so với lưu mảng số thô. Trường `term_freq_json` lưu sẵn term frequency dạng JSON string cùng với `doc_length`, để Luồng 2 chỉ cần tính cosine similarity + BM25 score rồi kết hợp bằng **Reciprocal Rank Fusion (RRF)** mà không cần gọi thêm engine tìm kiếm nào khác. Toàn bộ pipeline không còn bước "Lưu Vectors → OpenSearch" như sơ đồ gốc — bước đó giờ chính là 2 lần ghi DynamoDB (parent chunks và child chunks) ở trang này.
{{% /notice %}}

#### Nội dung tiếp theo

Tiếp theo: [5.3.5 - Cơ chế Resume OCR và xử lý lỗi](../5.3.5-Resume-OCR-Error-Handling/)
