---
title: "Retrieval Algorithms"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.8.1 </b> "
---

{{% notice note %}}
📌 All constants on this page (1200/250/40 tokens, `RRF_K=60`, `top_k=10`, `candidates_per_side=20`, `k1=1.5`, `b=0.75`) are configurable constants defined in code.
{{% /notice %}}

#### Parent-Child Chunking (`chunking.py`)

`modules/ingestion/lambda_src/document_processor/chunking.py`

Chunk sizes are calculated in **characters**, not tokens — avoiding the need to pull in a real tokenizer. The conversion factor `CHARS_PER_TOKEN = 3` is chosen conservatively (Vietnamese and non-Latin languages pack fewer characters/token than English), so **actual token counts are always ≤ target**, never exceeding embedding model input limits.

| Chunk Type | Constant | Target | Notes |
| --- | --- | --- | --- |
| Parent | `PARENT_TARGET_TOKENS` | 1200 tokens (≈3600 chars) | Prefers splitting at paragraph → sentence → line boundaries, hard splitting only when no natural boundary is found |
| Child | `CHILD_TARGET_TOKENS` | 250 tokens | — |
| Overlap | `CHILD_OVERLAP_TOKENS` | 40 tokens | Between adjacent child chunks — ensuring a sentence spanning a split boundary remains retrievable from both sides |

Only **child** chunks are embedded; **parent** chunks preserve raw text to serve as LLM context (reason explained in [Stream 1, page 5.3.4](../../5.3-Data-Ingestion/5.3.4-Chunking-Embedding/)).

#### Embedding — Abstraction via 2 API Shapes (`embeddings.py`)

`modules/ingestion/lambda_src/document_processor/embeddings.py`

The embedding model **can be swapped via environment variables without code changes**, because `embed_texts` automatically detects and handles the request/response shape of both supported Bedrock model families:

| Model | Invocation Style |
| --- | --- |
| **Cohere Embed** (default) | Invoked in batches of up to 96 texts/call, using `input_type` (`search_document` at ingestion / `search_query` at query time) to place questions and documents into an asymmetric vector space — improving retrieval accuracy |
| **Amazon Titan Embed** | Invoked text-by-text, without distinguishing `input_type` |

{{% notice note %}}
Cohere Embed is default **because Titan Embeddings was not available in `ap-southeast-1`** at the time of deployment — a practical regional constraint, not a purely technical choice.
{{% /notice %}}

#### Vector Packing (`vector_store.py`)

`modules/query/lambda_src/chat_engine/vector_store.py`

Vectors are **L2-normalized before storage**, so that during query time, cosine similarity between 2 normalized vectors reduces to a **pure dot product** — eliminating the need for square roots or division during every comparison.

Stored as `array("f").tobytes()` (Binary attribute in DynamoDB) instead of `List<Number>` — while both take 4KB for a 1024-dimensional vector, reading back costs only **1 `array.frombytes()` call** instead of deserializing 1024 individual `Decimal` values on every `Scan`.

```python
def normalize(vector):
    """L2-normalize a vector. Returns the zero vector unchanged (a
    zero-length embedding would only happen for degenerate/empty input)."""
    norm = math.sqrt(sum(x * x for x in vector))
    if norm == 0:
        return list(vector)
    return [x / norm for x in vector]


def pack_vector(vector):
    """Normalize and pack a vector of floats into bytes for storage."""
    return array.array("f", normalize(vector)).tobytes()


def unpack_vector(raw_bytes):
    """Unpack stored bytes back into a plain list of floats."""
    arr = array.array("f")
    arr.frombytes(bytes(raw_bytes))
    return arr.tolist()
```

#### Minimalist BM25 (`bm25.py`)

`modules/query/lambda_src/chat_engine/bm25.py`

Implements **standard Okapi BM25** (`k1=1.5, b=0.75`), with 2 **intentional** design choices:

- **Lucene-style IDF** (`log(1 + ...)`) instead of original Robertson/Sparck-Jones formula — avoiding negative IDF when a term appears in more than half the corpus, a common occurrence with **small corpora**.
- **Simple Tokenization** (`\w+` Unicode, lowercase only, no Vietnamese word segmentation, no stemming).

{{% notice tip %}}
Simple tokenization is **not an oversight**: exact surface-word matching is the exact goal of this lexical (BM25) branch — product codes, proper nouns, specific keywords. The "semantic understanding" (synonyms, paraphrasing) is already handled by the cosine branch.
{{% /notice %}}

```python
_TOKEN_PATTERN = re.compile(r"\w+", re.UNICODE)

# Standard constants from the Okapi BM25 literature. k1 controls how much
# additional occurrences of a term keep adding to its score (diminishing
# returns); b controls how much longer documents are penalized relative to
# the corpus average.
K1 = 1.5
B = 0.75


def tokenize(text):
    """Lowercase word tokens.

    `\\w+` with UNICODE matches Vietnamese letters (including diacritics) as
    ordinary word characters, so "việc" and "viec" tokenize as different,
    unrelated terms — a query must use matching diacritics to hit on exact
    terms. This only affects the lexical/BM25 side; cosine similarity from
    the embedding still catches the semantic match regardless of accents.
    """
    return _TOKEN_PATTERN.findall(text.lower())


def term_frequencies(tokens):
    freqs = {}
    for token in tokens:
        freqs[token] = freqs.get(token, 0) + 1
    return freqs


def idf(corpus_size, document_frequency):
    """Lucene-style IDF: always >= 0.

    The classic Robertson/Sparck-Jones formula (log((N-df+0.5)/(df+0.5)))
    can go negative for a term present in more than half the corpus, which
    would make that term's contribution to BM25 actively *subtract* from a
    document's score. That is standard IR behaviour for a huge corpus, but
    for the small, fast-changing document set this project targets it's an
    easy-to-hit, hard-to-explain edge case — adding 1 inside the log (as
    Lucene does) keeps every term's contribution non-negative instead.
    """
    return math.log(1 + (corpus_size - document_frequency + 0.5) / (document_frequency + 0.5))


def score(query_tokens, doc_term_freq, doc_length, avg_doc_length, corpus_size, document_frequencies):
    """BM25 score of one document against a tokenized query.

    `document_frequencies` maps each query term to how many documents in
    the scanned corpus contain it at least once — the caller computes this
    once per query (not once per document) since it's the same for every
    document being scored.
    """
    if doc_length <= 0 or avg_doc_length <= 0:
        return 0.0

    total = 0.0
    for term in query_tokens:
        tf = doc_term_freq.get(term, 0)
        if tf == 0:
            continue
        df = document_frequencies.get(term, 0)
        if df == 0:
            continue
        numerator = tf * (K1 + 1)
        denominator = tf + K1 * (1 - B + B * doc_length / avg_doc_length)
        total += idf(corpus_size, df) * (numerator / denominator)
    return total
```

#### RRF Fusion in a Single Scan (`retrieval.py`)

`modules/query/lambda_src/chat_engine/retrieval.py`

{{% notice note %}}
📌 **Most critical design point of the entire retrieval layer:** both cosine and BM25 are computed from **a single `Scan` of `child_chunks`**, not 2 scans. BM25 requires corpus-wide statistics (document frequency of query terms, average chunk length) — normally fetched from a search engine inverted index, here computed "piggybacked" alongside data reading for the cosine branch, so hybrid search **adds zero network round-trips** compared to a vector-only design.
{{% /notice %}}

**Why use Reciprocal Rank Fusion (`RRF_K = 60`) instead of weighted score summation?** Cosine similarity (bounded in `[-1,1]`) and BM25 scores (unbounded, corpus-dependent) exist on **2 incompatible scales** — weighted summation requires corpus-specific tuning to be meaningful. RRF ignores score magnitudes and fuses **rankings** only, requiring no tuning and avoiding scale skew.

Each branch retains up to `candidates_per_side = 20` top candidates before fusion, with final results truncated to `top_k = 10`.

{{% notice warning %}}
📌 **No minimum score threshold exists at this layer** — even if the best chunk has a very low cosine score (nearly irrelevant), it is still returned if it falls within the top-K. **The sole safety net** for completely off-topic questions resides in the **LLM system prompt** (_"if the context does not contain the answer, say plainly that you don't have enough information"_ — refer back to [Stream 2, page 5.4.4](../../5.4-Realtime-QA/5.4.4-Hybrid-Search-Retrieval/)), **not in the retrieval layer**.
{{% /notice %}}

```python
RRF_K = 60

_PROJECTION = (
    "chunk_id, parent_id, document_id, chunk_index, child_text, "
    "embedding, term_freq_json, doc_length"
)

def _scan_all_chunks(child_table):
    items = []
    scan_kwargs = {"ProjectionExpression": _PROJECTION}
    while True:
        response = child_table.scan(**scan_kwargs)
        items.extend(response.get("Items", []))
        if "LastEvaluatedKey" not in response:
            break
        scan_kwargs["ExclusiveStartKey"] = response["LastEvaluatedKey"]
    return items


def _reciprocal_rank_fusion(ranked_lists):
    scores = {}
    for ranking in ranked_lists:
        for rank, chunk_id in enumerate(ranking):
            scores[chunk_id] = scores.get(chunk_id, 0.0) + 1.0 / (RRF_K + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)


def hybrid_search(child_table, query_vector, query_tokens, top_k=10, candidates_per_side=20):
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


def fetch_parent_contexts(parent_table, hits, max_parents=4):
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

---

#### Next Content

- [5.8.2 - Semantic Cache & Observability](../5.8.2-Cache-and-Observability/)
