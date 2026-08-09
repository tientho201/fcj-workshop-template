---
title: "End-to-End Testing"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.3.6 </b> "
---
After completing the infrastructure ([5.3.1](../5.3.1-Infrastructure-S3-SQS/), [5.3.2](../5.3.2-Infrastructure-DynamoDB-IAM/)) and processing logic ([5.3.3](../5.3.3-Text-Extraction/), [5.3.4](../5.3.4-Chunking-Embedding/), [5.3.5](../5.3.5-Resume-OCR-Error-Handling/)), the final step is to test the entire end-to-end flow.

#### Test Scenarios

Upload files sequentially to test each processing branch:

1. **Plain text PDF file** → read immediately by `pypdf`, no OCR required.
2. **Scanned PDF file** → status transitions to `awaiting_ocr_confirmation`, invoke `/documents-decision` to confirm OCR, then processed by Textract (see [5.3.3](../5.3.3-Text-Extraction/) and [5.3.5](../5.3.5-Resume-OCR-Error-Handling/)).
3. **Image file (.png/.jpg)** → always OCR immediately without prompting.
4. **.txt/.md file** → decode UTF-8 directly.

After each upload, verify in the following order:

**SQS** (message consumed) → **CloudWatch Logs** (no errors, or `_report_to_function_dlq` logs if bugs exist) → **`ingestion_status` table** (final status = `completed`) → **`parent_chunks`/`child_chunks` tables** (new items created).

![End-to-end test results across all 4 file types](../images/11-end-to-end-test-result.png)
*Illustration: `ingestion_status` table showing 4 documents in `completed` status, with corresponding item count increases in `child_chunks`.*

#### Key Outcomes Achieved

- Fully event-driven ingestion pipeline via S3 → SQS without needing an intermediate Lambda function to receive events.
- Clear separation of error types (corrupt file vs. code bug) using 2 distinct DLQ tiers, speeding up debugging.
- Supports all 3 document types (plain text, PDFs with/without embedded text layers, scanned images) with an active OCR confirmation workflow instead of blindly OCR-ing all PDFs (reducing Textract costs).
- Vectors and BM25 statistics stored efficiently in DynamoDB (Binary + JSON string), eliminating the operational overhead of a separate OpenSearch Serverless cluster.
- Least-privilege IAM Role adherence, explicitly documenting exceptions (Textract) to prevent misunderstandings during security reviews.

#### Pipeline 1 Completion Checklist

- [ ] S3 bucket `raw_documents` has versioning enabled, SSE-S3 encryption, and 90-day Glacier lifecycle rule
- [ ] SQS `ingestion_queue` + 2 DLQs (`ingestion_dlq`, `document_processor_fn_dlq`) working as expected with `batch_size = 1`
- [ ] 3 DynamoDB tables (`parent_chunks`, `child_chunks`, `ingestion_status`) created
- [ ] IAM Role `document_processor` scoped strictly to 4 least-privilege permission statements
- [ ] PDF/image/text processing branches operating properly; `awaiting_ocr_confirmation` workflow testable via `/documents-decision`
- [ ] Direct invoke entrypoints (`resume_ocr`/`cancel`) called from `chat_engine` functioning correctly
- [ ] End-to-end testing verified across all 4 file types, with data correctly populated in DynamoDB

{{% notice warning %}}
Since Pipeline 1 no longer uses OpenSearch, **Pipeline 2 (Realtime Q&A)** must also be updated: instead of querying OpenSearch, it will directly read `child_chunks` from DynamoDB, calculate cosine similarity with the query vector + BM25 score, and combine them using RRF.
{{% /notice %}}
