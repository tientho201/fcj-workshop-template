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

##### Figure 1: End-to-end test results with PDF file

![End-to-end test results with PDF file](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-1.png)

##### Figure 2: End-to-end test results with TXT file

- For TXT files, the document text is extracted directly in the dialog box so you can click "Upload & Process".

![End-to-end test results with TXT file](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-2.png)

##### Figure 3: End-to-end test results with PNG file

- For PNG files, the file is sent in raw format (base64) for `document-processor` to run Textract OCR — text content cannot be displayed directly in the upload dialog box.

![End-to-end test results with PNG file](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-3.png)

##### Figure 4: End-to-end test results with MD file

- For MD files, the document text is extracted directly in the dialog box so you can click "Upload & Process".

![End-to-end test results with MD file](/images/5-Workshop/5.3-Data-Ingestion/image5.3.6-4.png)

#### Key Outcomes Achieved

- Fully event-driven ingestion pipeline via S3 → SQS without needing an intermediate Lambda function to receive events.
- Clear separation of error types (corrupt file vs. code bug) using 2 distinct DLQ tiers, speeding up debugging.
- Supports all 3 document types (plain text, PDFs with/without embedded text layers, scanned images) with an active OCR confirmation workflow instead of blindly OCR-ing all PDFs (reducing Textract costs).
- Vectors and BM25 statistics stored efficiently in DynamoDB (Binary + JSON string), eliminating the operational overhead of a separate OpenSearch Serverless cluster.
- Least-privilege IAM Role adherence, explicitly documenting exceptions (Textract) to prevent misunderstandings during security reviews.
