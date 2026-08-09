---
title: "End-to-End Testing"
date: 2026-08-07
weight: 4
chapter: false
pre: " <b> 5.7.4. </b> "
---

{{% notice note %}}
📌 This page is a **test scenario frame** aligned with the “things to try” list in `ui/README.md`. Fill in real timings and screenshots when you run the suite against a live stack.
{{% /notice %}}

After [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)–[5.7.3](../5.7.3-Deployment-Hosting/), open `ui/index.html`, paste connection fields from `terraform output`, and sign in with a Cognito user.

#### UI test scenarios

| # | Scenario | Expected |
|---|---|---|
| 1 | Login with a valid Cognito user | Receive `IdToken`; auth indicator turns green; Upload / Ask enabled |
| 2 | Login with wrong password | Clear error; UI does not crash; buttons stay disabled |
| 3 | Upload `.txt` / `.md` (text) | Content appears in the editor and can be edited before send |
| 4 | Upload a PDF that already has a text layer | Ingest completes without OCR confirm; extract via `pypdf` |
| 5 | Upload a scanned PDF (no text) — choose **Yes** | OCR confirm dialog → Textract → pipeline continues |
| 6 | Upload a scanned PDF — choose **No** | Stops as cancelled; no Textract cost |
| 7 | Ask about the document just uploaded | Grounded answer; source file tags when present |
| 8 | Ask the **same** question again (**same** session) | `cache hit` tag; mostly one lit step; response &lt; 1s |
| 9 | Vague follow-up in the **same** session (“còn cái kia thì sao?”) | Rewrite step lights; log shows rewritten query |
| 10 | Ask something not in the corpus | Model refuses / stays grounded instead of inventing freely |
| 11 | Burst many chat requests quickly | Possible 429 with `⏳` (retryable) — wait a few seconds and retry |

![Test results for the 11 scenarios with real screenshots](../images/07-test-scenarios-real-run.png)
*(Replace with your own screenshots after running all 11 scenarios.)*

{{% notice tip %}}
If ingest hangs past ~90s, check CloudWatch Logs for `document-processor` and SQS/DLQ depth (Stream 3 alarms). The UI timeout is intentional so a stuck pipeline surfaces instead of spinning forever.
{{% /notice %}}

#### Checklist before calling it “done”

- [ ] Token expired (wait &gt; 60 minutes) → next action shows a clear error, does not hang
- [ ] Binary upload &gt; ~7 MB → confirm you see the API Gateway 10 MB body limit message
- [ ] Reload the page while polling `/status` → no crash; connection config still present via `localStorage`
- [ ] Responsive: shrink below 900px → layout becomes one column
- [ ] `terraform output` values pasted into **1 · Kết nối**; password not left in `localStorage`
- [ ] Clear that frontend is local-only (no Amplify / S3 website in this stack)

{{% notice tip %}}
Scenario #11 (429) and the 10 MB body-limit check are the easiest to skip in a casual demo — both are real system limits worth recording in a “test results” section.
{{% /notice %}}

#### Outcomes

- Single-file UI exercises all four Stream 2 routes with a real Cognito JWT.
- Upload path distinguishes text vs binary (base64) for Textract.
- Pipeline animation reflects server `trace` / status timings, including skipped steps.
- Cache hit and multi-turn rewrite are visible without opening the AWS Console.
