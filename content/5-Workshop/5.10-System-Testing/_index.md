---
title: "System Testing"
date: 2024-01-01
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---

#### Backend Test Scenarios

| #   | Scenario                                                      | Expected Outcome                                                                                                        |
| --- | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| 1   | `terraform fmt -check` + `terraform validate`                 | Pass — runs automatically in CI (`ci.yml`) on every PR                                                                  |
| 2   | `ruff check modules`                                          | Pass — lints Lambda code, excluding vendored `pypdf`                                                                    |
| 3   | Ingest plain text file                                        | Direct text extraction, bypassing Textract                                                                              |
| 4   | Ingest PDF with text layer                                    | Readable by `pypdf`, incurring no Textract costs                                                                        |
| 5   | Ingest scanned PDF                                            | Pauses at `awaiting_ocr_confirmation`; confirming Yes → Textract OCR runs correctly, confirming No → status `cancelled` |
| 6   | Re-upload the same file                                       | Old chunks deleted before re-indexing, preventing duplicate data                                                        |
| 7   | Ask semantically equivalent question with different words     | Cosine captures it (vector branch), even when BM25 matches no words                                                     |
| 8   | Ask for exact code/proper name present in document            | BM25 pulls exact chunk containing that term to top, even if cosine ranks lower                                          |
| 9   | Call `/chat` twice with identical question, different session | Request 2 does **not** hit request 1's cache if request 1's session has history (cache bypassed when history exists)    |
| 10  | Remove `lambda` VPC endpoint then call `/documents-decision`  | Reproduces exact 504 error encountered during real development — confirming endpoint issue, not code bug                |
| 11  | Send rapid concurrent questions                               | Bedrock returns `ThrottlingException`, backend returns `429 {retryable:true}`, Lambda does not crash                    |

![CI run results (fmt/validate/lint) and manual test scenarios above](../images/05-backend-test-results.png)
_Illustration: `terraform validate`, `ruff check` results, and CloudWatch logs for test scenarios 7-8 (cosine vs BM25)._

#### Real Tests Conducted During Development (Not Assumptions)

{{% notice note %}}
📌 Items **5, 6, and 10** above are actual bugs/scenarios that **occurred and were fixed** during project development history:

- **Missing VPC endpoint bug** for Lambda control-plane API caused 504 errors when `chat_engine` directly invoked `document_processor` — discovered during real E2E testing, fixed by adding `aws_vpc_endpoint "lambda"` (`modules/networking/main.tf`).
- **OCR confirmation flow** (item 5) was tested on both branches (accept/reject) using a manually crafted image-only PDF (no text layer), confirming correct pause behavior — neither silently running OCR nor silently ignoring the document.
  {{% /notice %}}

#### Untested Automations (Gaps to Highlight in Report)

{{% notice note %}}
📌 **Update:** see [System Testing, page 5.10.3](../../5.10-Testing/5.10.3-Khoang-trong-va-De-xuat/): 33 unit tests (`pytest`) written for the exact modules mentioned above (`chunking.py`, `bm25.py`, `retrieval.py`, `vector_store.py`), running automatically in CI. Remaining **gap**: `handler.py` for both Lambdas (code making direct `boto3`/AWS calls) still lacks automated unit testing.
{{% /notice %}}

#### Detailed Contents

1. [Layer 1 — Automated Static Analysis (CI)](5.10.1-Layer-1-Automated-Static-Analysis/)
2. [Layer 2 — Manual/E2E Testing](5.10.2-Layer-2-Manual-E2E-Testing/)
3. [Layer 3 — Automated Unit Tests for Pure Logic](5.10.3-Layer-3-Automated-Unit-Testing/)
