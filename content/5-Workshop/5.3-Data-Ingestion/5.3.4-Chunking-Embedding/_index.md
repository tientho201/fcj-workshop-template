---
title: "Parent-Child Chunking and Embedding"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.3.4 </b> "
---

After extracting text (from page [5.3.3](../5.3.3-Text-Extraction/)), the next step is to chunk the document, generate vector embeddings, and store them into the 2 DynamoDB tables created on page [5.3.2](../5.3.2-Infrastructure-DynamoDB-IAM/).

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

Text is split into 2 parallel types of chunks:

- **Parent chunk** (~1000-1500 tokens) — serves purely as context for the LLM when generating answers; it is not embedded.
- **Child chunk** (~200-300 tokens) — will be embedded and used for precise vector retrieval.

Before writing new data, Lambda **deletes all old chunks matching the same `document_id`** (if a user re-uploads the same key) to prevent data duplication upon re-ingestion.

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

#### Storing Parent Chunks in DynamoDB

```python
def _store_parents(parents):
    with parent_table.batch_writer() as batch:
        for parent in parents:
            batch.put_item(Item=parent)
```

Only raw text is saved — serving broad context retrieval by `parent_id` during answer generation in Pipeline 2.

#### Embedding Child Chunks via Amazon Bedrock

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

The embedding model is retrieved from the `BEDROCK_EMBEDDING_MODEL_ID` environment variable, matching the exact ARN scoped in the IAM Role on page 5.3.2 — ensuring Lambda can only invoke this specific model and no other Bedrock models.

#### Storing Child Chunks: Binary Vectors + BM25 Term Frequency

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
`vector_store.pack_vector()` packs vectors into a **Binary attribute** (`embedding`) instead of a `List<Number>` — saving significant storage cost and read/write bandwidth on DynamoDB compared to raw numerical arrays. The `term_freq_json` field stores pre-computed term frequencies as a JSON string alongside `doc_length`, allowing Pipeline 2 to simply calculate cosine similarity + BM25 score and combine them using **Reciprocal Rank Fusion (RRF)** without invoking an external search engine. The entire pipeline no longer includes a "Store Vectors → OpenSearch" step as in the original diagram — that step now consists of the 2 DynamoDB writes (parent chunks and child chunks) on this page.
{{% /notice %}}

---

#### Next content

Next: [5.3.5 - Resume OCR Mechanism and Error Handling](../5.3.5-Resume-OCR-Error-Handling/)
