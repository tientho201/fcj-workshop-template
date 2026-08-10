---
title: "Manual E2E Testing"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.10.1 </b> "
---

{{% notice note %}}
📌 This section describes events that **actually occurred** during development, **not speculation** — but executed via temporary scripts in a scratchpad, **not committed to the repository**, and thus not automatically repeatable. Distinct from static checks in CI ([5.9.1](../../5.9-CICD/5.9.1-CI-Workflow/)) or automated unit tests ([5.8.4](../../5.8-Backend/5.8.4-Backend-Testing/)) — both of which re-run identically every time — the content of this page represents **point-in-time evidence**, not a reusable process.
{{% /notice %}}

#### Hand-Crafted 3 PDF Test Types — Covering 3 Logic Branches

To accurately test all 3 branching text extraction paths of `_process_s3_object` (described in [Stream 1, page 5.3.3](../../5.3-Data-Ingestion/5.3.3-Text-Extraction/)) — rather than just testing "runs without errors" — 3 distinct PDF types were manually constructed:

| PDF Type | Construction Method | Logic Branch Tested |
| --- | --- | --- |
| **PDF with embedded text layer** | Hand-writing individual PDF objects/xref/trailer **at byte level** — as the machine lacked pre-installed `reportlab`/`fpdf` | `pypdf` reads text directly, requiring no OCR |
| **PDF containing only images, no text** | `PIL.Image.save(..., "PDF")` | `pypdf` returns empty string → transitions to `awaiting_ocr_confirmation` |
| **PDF with text drawn inside the image** | `PIL.ImageDraw`/`ImageFont` rendering text onto image before saving to PDF | Textract has **actual content** to OCR out, not just blank/noisy images |

```python
# Example constructing Type 3 PDF (image with real text) - illustrating approach
from PIL import Image, ImageDraw, ImageFont

img = Image.new("RGB", (800, 1000), color="white")
draw = ImageDraw.Draw(img)
draw.text((50, 50), "This is real OCR test content", fill="black")
img.save("test-scanned-with-real-text.pdf", "PDF")
```

{{% notice tip %}}
**Hand-writing PDF objects/xref/trailer at the byte level** for Type 1 PDFs is the most notable detail — demonstrating a deep understanding of the PDF file format (beyond just using existing libraries), necessitated because the development environment had no pre-installed `reportlab`/`fpdf`. This is a technical highlight worth including in reports as proof of problem-solving capability when standard tooling is unavailable.
{{% /notice %}}

#### Real API Invocation Script, No Mocks

A custom Python script (using `requests`) **reimplemented the exact `cognitoLogin()` flow** used by the UI (`InitiateAuth` with `USER_PASSWORD_AUTH` — refer back to [Frontend, page 5.7.1](../../5.7-Frontend/5.7.1-Frontend-Architecture-Authentication/)), used to test `/documents`, `/status`, `/documents-decision`, `/chat` **end-to-end on real AWS infrastructure**, mocking no components.

```python
# Illustrating real test script structure (not an official test suite)
def cognito_login(username, password):
    response = requests.post(
        f"https://cognito-idp.{region}.amazonaws.com/",
        headers={"X-Amz-Target": "AWSCognitoIdentityProviderService.InitiateAuth", ...},
        json={"AuthFlow": "USER_PASSWORD_AUTH", ...},
    )
    return response.json()["AuthenticationResult"]["IdToken"]

token = cognito_login(test_user, test_password)
upload_result = requests.post(f"{api_url}/documents", headers={"Authorization": token}, json={...})
# ... continue testing /status, /documents-decision, /chat
```

#### Real Bug Caught by Testing — Not Speculation

{{% notice warning %}}
📌 **This is the most concrete evidence of the value of manual testing:** invoking `/documents-decision` with `decision: "ocr"` initially **returned `504`** — not a hypothetical bug. Tracing this error led directly to discovering a **missing VPC endpoint for the Lambda control-plane API** (detailed in [Stream 2, page 5.4.6](../../5.4-Realtime-QA/5.4.6-Error-Handling-OCR-Decision/) and [Backend, page 5.8.4](../../5.8-Backend/5.8.4-Backend-Testing/)).
{{% /notice %}}

After adding `aws_vpc_endpoint "lambda"` and re-applying, **re-testing confirmed the complete OCR resume flow executed end-to-end**:

```
confirm → Textract OCR → chunk → embed → index → completed
```

This serves as the clearest example in the entire project showing that **manual testing, even when unautomated, retains critical value for catching real bugs** — without this end-to-end test pass on real infrastructure, the missing VPC endpoint bug might have gone undetected until hit by actual end users.

---
