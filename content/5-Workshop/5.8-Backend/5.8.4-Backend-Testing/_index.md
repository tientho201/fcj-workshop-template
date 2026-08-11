---
title: "Backend Testing"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.8.4 </b> "
---

#### Existing: Automated Test Suite for Pure Python Logic

`tests/` (**33 test cases**, executed via `pytest`, connected to CI via the `python-test` job in `ci.yml` — see [5.9.1](../../5.9-CICD/5.9.1-CI-Workflow/)) — no AWS mocks, testing only the logic separable from `boto3`:

| File                            | Test Count | What is Tested                                                                                                                                                                           |
| ------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test_chunking.py`              | 8          | Parent/child chunking: empty chunks, `parent_id` format, CRLF normalization, paragraph boundary splitting priority, monotonically increasing `chunk_index`, overlap between child chunks |
| `test_bm25.py`                  | 9          | Tokenization distinguishing Vietnamese diacritics, non-negative IDF, rewarding multiple matching terms, penalizing long documents                                                        |
| `test_vector_store.py`          | 6          | L2-normalization, round-trip pack/unpack preserving float32 precision, dot product                                                                                                       |
| `test_retrieval.py`             | 6          | Hand-calculation verifying exact RRF formula; constructing `FakeChildTable`/`FakeParentTable` mocking `.scan()`/`.batch_get_item()` to prove `hybrid_search` is genuinely "hybrid"       |
| `test_shared_copies_in_sync.py` | 4          | Verifying drift between manual duplicate copies of `bm25.py`/`vector_store.py`/`tracing.py`/`embeddings.py` across both Lambdas                                                          |

{{% notice tip %}}
**Notable technical detail on import mechanics:** `modules/*/lambda_src/*/` **are not Python packages** (no `__init__.py`, Lambda zips each directory flat) — tests use `importlib.util.spec_from_file_location` to **load directly by file path** instead of standard `import`, preventing same-named files (`bm25.py` existing in both Lambdas) from overwriting each other in a single test process.

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

#### Completed: Manual/Real E2E Testing During Development (Not Automatically Repeatable)

Full details on [5.10.1](../../5.10-System-Testing/5.10.1-Manual-E2E-Testing/), summarized:

- Manually constructed 3 PDF types (with text layer / image-only / image with rendered text) to test all 3 extraction branches of `document_processor`.
- Invoked real APIs (Cognito `InitiateAuth` + `requests`) to test end-to-end `/documents`, `/status`, `/documents-decision`, `/chat`.
- Discovered and fixed a real bug: missing VPC endpoint for Lambda control-plane API causing `504` errors when `chat_engine` directly invoked `document_processor`.

#### Remaining Gaps

{{% notice warning %}}

- **No tests for `handler.py`** in either Lambda (parts directly calling `boto3` — S3, DynamoDB, Bedrock, Textract) — requires mocks (`moto` or similar) or dedicated integration tests to cover further.
- **No tests for `/feedback` route** or the join between `evaluation_runner.py` ↔ `message_id-index` (noted on [5.8.2](../5.8.2-Cache-and-Observability/) and [Stream 4, page 5.6.3](../../5.6-RAGAS/5.6.3-RAGAS-Evaluation-Logic/)) — logic has been manually cross-checked (field/GSI names match between Terraform and Python) but **lacks automated tests**.
  {{% /notice %}}
