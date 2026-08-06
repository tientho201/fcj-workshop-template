---
title: "Hybrid Search and Retrieval"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.4.4 </b> "
---

Following the cache lookup and query rewriting steps on page [5.4.3](../5.4.3-Cache-Lookup-Query-Rewriting/), this page presents the core part of RAG: question moderation, relevant context retrieval, and fetching full parent context content.

#### Step 3 — Input Guardrail / Output Guardrail

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

The raw question (after rewriting if needed) is evaluated via Bedrock Guardrail (configured on page [5.4.2](../5.4.2-Cache-Guardrails-IAM/)) **before** entering the embedding/retrieval step — early blocking policy-violating questions right at the start of the pipeline.

#### Step 4 — Embed Question and Hybrid Search

```python
def hybrid_search(child_table, query_vector, query_tokens, top_k=10, candidates_per_side=20):
    """Scan the child-chunks table once and return the top_k chunks by
    fused cosine + BM25 rank.

    `query_tokens` should come from bm25.tokenize(rewritten_question) — a
    pre-tokenized query, not raw text, since the caller (handler.py) already
    needs the raw question for other purposes and tokenizing twice would be
    wasteful.

    Returns a list of dicts: {parent_id, document_id, chunk_index,
    child_text}, ordered by descending fused rank. (No numeric "score" field:
    RRF fuses *rankings*, not comparable magnitudes, so a single fused score
    wouldn't mean anything on its own — the ordering is the output.)
    """
    items = _scan_all_chunks(child_table)
    if not items:
        return []

    query_unit = vector_store.normalize(query_vector)

    # --- Cosine similarity, one pass over the already-fetched items. ---
    cosine_scores = {}
    for item in items:
        vector = vector_store.unpack_vector(item["embedding"])
        cosine_scores[item["chunk_id"]] = vector_store.dot(query_unit, vector)
    cosine_ranking = sorted(cosine_scores, key=cosine_scores.get, reverse=True)[:candidates_per_side]

    # --- BM25, using corpus statistics derived from this same Scan. ---
    bm25_ranking = []
    if query_tokens:
        corpus_size = len(items)
        total_length = 0
        document_frequencies = {term: 0 for term in query_tokens}
        parsed = []  # (chunk_id, term_freq, doc_length)

        for item in items:
            term_freq = json.loads(item.get("term_freq_json") or "{}")
            doc_length = int(item.get("doc_length") or 0)
            total_length += doc_length
            parsed.append((item["chunk_id"], term_freq, doc_length))
            for term in query_tokens:
                if term_freq.get(term):
                    document_frequencies[term] += 1

        avg_doc_length = (total_length / corpus_size) if corpus_size else 0

        bm25_scores = {
            chunk_id: bm25.score(
                query_tokens, term_freq, doc_length, avg_doc_length, corpus_size, document_frequencies
            )
            for chunk_id, term_freq, doc_length in parsed
        }
        # Only worth ranking on BM25 if at least one chunk actually matched a
        # query term — otherwise every score is 0 and the "ranking" would be
        # arbitrary Python dict order, which would corrupt the RRF fusion
        # below with meaningless positions.
        if any(bm25_scores.values()):
            bm25_ranking = sorted(bm25_scores, key=bm25_scores.get, reverse=True)[:candidates_per_side]

    fused_ids = _reciprocal_rank_fusion([r for r in (cosine_ranking, bm25_ranking) if r])[:top_k]

    items_by_id = {item["chunk_id"]: item for item in items}
    return [
        {
            "chunk_id": chunk_id,
            "parent_id": items_by_id[chunk_id].get("parent_id"),
            "document_id": items_by_id[chunk_id].get("document_id"),
            "chunk_index": items_by_id[chunk_id].get("chunk_index"),
            "child_text": items_by_id[chunk_id].get("child_text"),
        }
        for chunk_id in fused_ids
    ]
```

{{% notice note %}}
Instead of invoking a dedicated search engine, the `retrieval.hybrid_search` function scans the `child_table` once, calculates **cosine similarity** (semantic similarity via vectors) and **BM25 score** (keyword matching) for each chunk, then fuses the two ranking results using the **Reciprocal Rank Fusion (RRF)** algorithm — outputting a balanced result list combining semantic understanding and exact keyword matching.
{{% /notice %}}

#### Step 5 — Fetch Parent Chunks as LLM Context

##### 1. The `fetch_parent_contexts` function (`retrieval.py:156-189`)

Collects unique parent_ids from the top hybrid search results (up to max_parents = 4) and uses `batch_get_item` to query DynamoDB for full parent chunk content:

```python
def fetch_parent_contexts(parent_table, hits, max_parents=4):
    """Resolve top hybrid-search hits to their parents' full text.

    Walks the fused ranking in order and collects distinct parent_ids, so
    parents come back in relevance order and a parent whose children
    dominate the ranking is only included once.
    """
    ordered_parent_ids = []
    for hit in hits:
        parent_id = hit.get("parent_id")
        if parent_id and parent_id not in ordered_parent_ids:
            ordered_parent_ids.append(parent_id)
        if len(ordered_parent_ids) >= max_parents:
            break

    if not ordered_parent_ids:
        return []

    # BatchGetItem returns items in arbitrary order, so index them and re-emit
    # in the relevance order established above.
    response = parent_table.meta.client.batch_get_item(
        RequestItems={
            parent_table.name: {
                "Keys": [{"parent_id": parent_id} for parent_id in ordered_parent_ids],
                "ProjectionExpression": "parent_id, document_id, parent_text",
            }
        }
    )
    items_by_id = {
        item["parent_id"]: item
        for item in response.get("Responses", {}).get(parent_table.name, [])
    }

    return [items_by_id[pid] for pid in ordered_parent_ids if pid in items_by_id]
```

##### 2. Invocation flow and no-context handling in `handler.py:545-569`

In `handler.py`, the above function is invoked and checked: if `parents` is empty, an immediate default message is returned while skipping the Claude (Bedrock) answer generation step — avoiding wasted model invocation tokens for a query that certainly lacks valid context to answer.

```python
    parents = retrieval.fetch_parent_contexts(parent_chunks_table, fused_hits)
    tracer.step(
        "parents",
        "Lấy parent chunk",
        detail=f"{len(parents)} parent — ngữ cảnh đầy đủ đưa cho LLM",
    )
    logger.info("Retrieved %d fused hit(s) -> %d parent chunk(s)", len(fused_hits), len(parents))

    if not parents:
        answer = (
            "Tôi không tìm thấy tài liệu nào liên quan để trả lời câu hỏi này. "
            "Hãy thử tải tài liệu lên trước, hoặc đặt câu hỏi khác."
        )
        _persist_turn(session_id, question, answer, [])
        tracer.skip("generate", "Sinh câu trả lời (Claude)", detail="Bỏ qua — không có ngữ cảnh")
        return _response(
            200,
            {
                "answer": answer,
                "session_id": session_id,
                "sources": [],
                "trace": tracer.to_dict(),
            },
        )
```

---

#### Next Content

- [5.4.5 - Answer Generation and History Storage](../5.4.5-Answer-Generation-History-Storage/)
