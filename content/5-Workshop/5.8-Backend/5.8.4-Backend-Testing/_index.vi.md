---
title: "Kiểm thử Backend"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.8.4 </b> "
---

#### Đã có: bộ test tự động cho logic thuần Python

`tests/` (**33 test case**, chạy bằng `pytest`, nối vào CI qua job `python-test` trong `ci.yml` — xem [5.9.1](../../5.9-CICD/5.9.1-CI-Workflow/)) — không mock AWS, chỉ test đúng phần logic tách được khỏi `boto3`:

| File                            | Số test | Test gì                                                                                                                                                        |
| ------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test_chunking.py`              | 8       | Parent/child chunking: chunk rỗng, format `parent_id`, chuẩn hóa CRLF, ưu tiên cắt tại ranh giới đoạn văn, `chunk_index` tăng liên tục, overlap giữa các child |
| `test_bm25.py`                  | 9       | Tokenize phân biệt dấu tiếng Việt, IDF không âm, thưởng nhiều từ khớp, phạt tài liệu dài                                                                       |
| `test_vector_store.py`          | 6       | L2-normalize, round-trip pack/unpack giữ độ chính xác float32, dot product                                                                                     |
| `test_retrieval.py`             | 6       | Tính tay đúng công thức RRF; dựng `FakeChildTable`/`FakeParentTable` giả lập `.scan()`/`.batch_get_item()` để chứng minh `hybrid_search` thật sự "hybrid"      |
| `test_shared_copies_in_sync.py` | 4       | Kiểm tra drift giữa 2 bản sao tay của `bm25.py`/`vector_store.py`/`tracing.py`/`embeddings.py` ở 2 Lambda                                                      |

{{% notice tip %}}
**Chi tiết kỹ thuật đáng chú ý về cách import:** `modules/*/lambda_src/*/` **không phải Python package** (không có `__init__.py`, Lambda zip phẳng từng thư mục) — test dùng `importlib.util.spec_from_file_location` để **nạp trực tiếp theo đường dẫn file** thay vì `import` chuẩn, tránh việc 2 file cùng tên (`bm25.py` tồn tại ở cả 2 Lambda) đè lên nhau trong cùng 1 process test.

```python
import importlib.util

def load_module(file_path, module_name):
    spec = importlib.util.spec_from_file_location(module_name, file_path)
    module = importlib.util.module_from_spec(spec)
    sys.modules[module_name] = module
    spec.loader.exec_module(module)
    return module
```

{{% /notice %}}

#### Đã kiểm thử tay/E2E thật trong lúc phát triển (không lặp lại tự động được)

Chi tiết đầy đủ ở [5.10.1](../../5.10-System-Testing/5.10.1-Manual-E2E-Testing/), tóm tắt lại:

- Dựng tay 3 loại PDF (có text layer / chỉ ảnh / ảnh có chữ vẽ) để test đúng 3 nhánh trích xuất của `document_processor`.
- Gọi API thật (Cognito `InitiateAuth` + `requests`) để test end-to-end `/documents`, `/status`, `/documents-decision`, `/chat`.
- Tìm ra và sửa 1 bug thật: thiếu VPC endpoint cho Lambda control-plane API gây `504` khi `chat_engine` gọi `document_processor` trực tiếp.

#### Còn thiếu

{{% notice warning %}}

- **Không có test cho `handler.py`** của cả 2 Lambda (phần gọi `boto3` trực tiếp — S3, DynamoDB, Bedrock, Textract) — cần mock (`moto` hoặc tương tự) hoặc integration test riêng nếu muốn phủ tiếp.
- **Không có test cho route `/feedback`** hay join `evaluation_runner.py` ↔ `message_id-index` (đã nói ở [5.8.2](../5.8.2-Cache-va-Observability/) và [Luồng 4, trang 5.6.3](../../5.6-RAGAS/5.6.3-RAGAS-Evaluation-Logic/)) — logic đã kiểm tra chéo thủ công (tên field/GSI khớp giữa Terraform và Python) nhưng **chưa có test tự động**.
  {{% /notice %}}
