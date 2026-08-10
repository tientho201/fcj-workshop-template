---
title: "Thuật toán Retrieval"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.8.1 </b> "
---

{{% notice note %}}
📌 Toàn bộ hằng số trong trang này (1200/250/40 token, `RRF_K=60`, `top_k=10`, `candidates_per_side=20`, `k1=1.5`, `b=0.75`)
{{% /notice %}}

#### Parent-Child Chunking (`chunking.py`)

`modules/ingestion/lambda_src/document_processor/chunking.py`

Kích thước chunk tính theo **ký tự**, không theo token — tránh phải kéo theo một tokenizer thật. Hệ số quy đổi `CHARS_PER_TOKEN = 3` được chọn thận trọng (tiếng Việt và các ngôn ngữ non-Latin đóng gói ít ký tự/token hơn tiếng Anh), nên **số token thật luôn ≤ mục tiêu**, không bao giờ vượt giới hạn input của model embedding.

| Loại chunk | Hằng số                | Mục tiêu                 | Ghi chú                                                                                                 |
| ---------- | ---------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------- |
| Parent     | `PARENT_TARGET_TOKENS` | 1200 token (≈3600 ký tự) | Ưu tiên cắt tại ranh giới đoạn văn → câu → dòng, chỉ cắt cứng khi không tìm được ranh giới tự nhiên nào |
| Child      | `CHILD_TARGET_TOKENS`  | 250 token                | —                                                                                                       |
| Overlap    | `CHILD_OVERLAP_TOKENS` | 40 token                 | Giữa các child liền kề — để một câu nằm vắt ngang ranh giới cắt vẫn truy xuất được từ cả 2 phía         |

Chỉ **child** được embed; **parent** giữ nguyên văn để làm ngữ cảnh cho LLM (lý do đã nêu ở [Luồng 1, trang 5.3.4](../../5.3-Data-Ingestion/5.3.4-Chunking-Embedding/)).

#### Embedding — trừu tượng hóa qua 2 shape API (`embeddings.py`)

`modules/ingestion/lambda_src/document_processor/embeddings.py`

Model embedding **đổi được qua biến môi trường mà không sửa code**, vì hàm `embed_texts` tự nhận diện và xử lý đúng shape request/response của cả 2 họ model Bedrock hỗ trợ:

| Model                       | Cách gọi                                                                                                                                                                                                       |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cohere Embed** (mặc định) | Gọi theo batch, tối đa 96 text/lần, dùng `input_type` (`search_document` lúc ingest / `search_query` lúc hỏi) để đặt câu hỏi và tài liệu vào không gian vector bất đối xứng — cải thiện độ chính xác truy xuất |
| **Amazon Titan Embed**      | Gọi từng text một, không phân biệt `input_type`                                                                                                                                                                |

{{% notice note %}}
Cohere Embed là mặc định **vì Titan Embeddings chưa có ở `ap-southeast-1`** tại thời điểm triển khai — một ràng buộc thực tế về region, không phải lựa chọn kỹ thuật thuần túy.
{{% /notice %}}

#### Đóng gói vector (`vector_store.py`)

`modules/query/lambda_src/chat_engine/vector_store.py`

Vector được **L2-normalize trước khi lưu**, để lúc truy vấn, cosine similarity giữa 2 vector đã chuẩn hóa rút gọn thành phép **dot product thuần** — không cần căn bậc hai hay phép chia mỗi lần so sánh.

Lưu dưới dạng `array("f").tobytes()` (Binary attribute trong DynamoDB) thay vì `List<Number>` — cùng là 4KB cho vector 1024 chiều, nhưng đọc lại chỉ tốn **1 lệnh `array.frombytes()`** thay vì deserialize 1024 giá trị `Decimal` riêng lẻ mỗi lần `Scan`.

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

#### BM25 tối giản (`bm25.py`)

`modules/query/lambda_src/chat_engine/bm25.py`

Cài đặt **Okapi BM25 chuẩn** (`k1=1.5, b=0.75`), với 2 điểm khác biệt **có chủ đích**:

- **IDF kiểu Lucene** (`log(1 + ...)`) thay vì công thức Robertson/Sparck-Jones gốc — tránh IDF âm khi một từ xuất hiện ở hơn nửa corpus, trường hợp dễ gặp với **corpus nhỏ**.
- **Tokenize đơn giản** (`\w+` Unicode, chỉ lowercase, không tách từ tiếng Việt, không stemming).

{{% notice tip %}}
Việc tokenize đơn giản **không phải thiếu sót**: so khớp chính xác bề mặt từ đúng là mục tiêu của nhánh lexical (BM25) này — mã sản phẩm, tên riêng, từ khóa cụ thể. Phần "hiểu nghĩa" (đồng nghĩa, diễn đạt khác) đã có nhánh cosine đảm nhiệm.
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

#### Hợp nhất bằng RRF, chỉ 1 lần Scan (`retrieval.py`)

`modules/query/lambda_src/chat_engine/retrieval.py`

{{% notice note %}}
📌 **Điểm thiết kế quan trọng nhất của toàn bộ retrieval:** cả cosine lẫn BM25 đều tính từ **đúng 1 lần `Scan` bảng `child_chunks`**, không phải 2 lần. BM25 cần thống kê toàn corpus (document frequency của từng từ trong query, độ dài trung bình chunk) — bình thường phải lấy từ inverted index của một search engine, ở đây được tính "ăn theo" ngay trong lúc quét dữ liệu cho nhánh cosine, nên hybrid search **không tốn thêm round-trip mạng nào** so với thiết kế chỉ-vector trước đó.
{{% /notice %}}

**Vì sao dùng Reciprocal Rank Fusion (`RRF_K = 60`) thay vì cộng điểm có trọng số?** Cosine similarity (bị chặn trong `[-1,1]`) và điểm BM25 (không bị chặn, phụ thuộc corpus) nằm trên **2 thang đo không tương thích** — cộng có trọng số cần tinh chỉnh riêng cho từng corpus mới có ý nghĩa. RRF bỏ qua độ lớn, chỉ hợp nhất **thứ hạng**, không cần tinh chỉnh và khó bị lệch.

Mỗi nhánh chỉ giữ lại tối đa `candidates_per_side = 20` ứng viên tốt nhất trước khi fuse, kết quả cuối cắt còn `top_k = 10`.

{{% notice warning %}}
📌 **Không có ngưỡng điểm tối thiểu nào ở tầng này** — dù chunk tốt nhất có cosine rất thấp (gần như không liên quan), nó vẫn được trả về nếu nằm trong top-K. **Tấm lưới an toàn duy nhất** cho câu hỏi hoàn toàn lạc đề nằm ở **prompt hệ thống của LLM** (_"nếu ngữ cảnh không có câu trả lời, hãy nói rõ là không đủ thông tin"_ — xem lại [Luồng 2, trang 5.4.4](../../5.4-Realtime-QA/5.4.4-Hybrid-Search-Retrieval/)), **không phải ở tầng retrieval**.
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

#### Nội dung tiếp theo

- [5.8.2 - Semantic Cache & Observability](../5.8.2-Cache-va-Observability/)
