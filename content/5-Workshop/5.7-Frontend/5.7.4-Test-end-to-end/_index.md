---
title: "End-to-End Testing"
date: 2026-08-07
weight: 4
chapter: false
pre: " <b> 5.7.4. </b> "
---

After [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)–[5.7.3](../5.7.3-Deployment-Hosting/), verify the UI against a live API. Open `ui/index.html`, fill connection fields from `terraform output`, and sign in with a Cognito user.

#### Test scenarios

| # | Action | Expected result |
|---|---|---|
| 1 | Login with valid Cognito user | Auth dot green; Upload and Ask enabled; ID token received |
| 2 | Login with wrong password | Error in log; buttons stay disabled |
| 3 | Upload a `.txt` / `.md` about a known topic, wait for ingest | Right pane shows ingest steps; summary with parent/child counts |
| 4 | Ask a question grounded in that document | Answer cites content; source tags when present; query flow animates |
| 5 | Click **Phiên mới**, ask the **same** question again | `cache hit` tag; mostly one lit step; sub-second response |
| 6 | Ask something not in the corpus | Model refuses / stays grounded instead of inventing freely |
| 7 | Follow-up vague question in the **same** session (“còn cái kia thì sao?”) | Query-rewriting step lights; log shows rewritten query |
| 8 | Upload a scanned / image-like PDF (no text layer) | `awaiting_ocr_confirmation` → Yes/No box → OCR or cancel path |
| 9 | Burst many chat requests | Possible HTTP 429 with `retryable` — wait and retry (expected under low Bedrock quota) |

{{% notice tip %}}
If ingest hangs past ~90s, check CloudWatch Logs for `document-processor` and SQS/DLQ depth (Stream 3 alarms). The UI timeout is intentional so a stuck pipeline surfaces instead of spinning forever.
{{% /notice %}}

#### Outcomes

- Single-file UI exercises all four Stream 2 routes with a real Cognito JWT.
- Upload path distinguishes text vs binary (base64) for Textract.
- Pipeline animation reflects server `trace` / status timings, including skipped steps.
- Cache hit and multi-turn rewrite are visible without opening the AWS Console.

#### Frontend completion checklist

- [ ] `terraform output` values pasted into **1 · Kết nối**
- [ ] Cognito login succeeds; password not left in `localStorage`
- [ ] Text upload completes and appears in chat answers
- [ ] Cache-hit path verified after **Phiên mới**
- [ ] OCR confirmation path understood (even if not demoed every time)
- [ ] Clear that frontend is local-only (no Amplify/S3 website in this stack)
