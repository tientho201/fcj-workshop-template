---
title: "Chat Interface and Document Upload"
date: 2026-08-07
weight: 2
chapter: false
pre: " <b> 5.7.2. </b> "
---

This page covers the left-pane upload and chat flows in `ui/index.html`, plus the right-pane pipeline animation that replays real backend timings.

#### Document upload — text vs binary

| Kind | Extensions | How the browser reads | Body sent to `POST /documents` |
|---|---|---|---|
| Text | `.txt`, `.md`, `.csv`, `.json`, `.log`, `.htm`, `.html` | `file.text()` | `{ filename, content }` |
| Binary (OCR) | `.pdf`, `.png`, `.jpg`, `.jpeg`, `.tiff`, `.tif` | `arrayBuffer()` → base64 | `{ filename, content_base64, content_type }` |

Binary files must not use `file.text()` — that would corrupt bytes before Textract. The editor becomes read-only and shows a placeholder; pick another file to change content.

```javascript
const BINARY_EXTENSIONS = new Set([".pdf", ".png", ".jpg", ".jpeg", ".tiff", ".tif"]);

if (state.binaryUpload) {
  uploadBody = {
    filename,
    content_base64: state.binaryUpload.base64,
    content_type: state.binaryUpload.contentType,
  };
} else {
  uploadBody = { filename, content: $("fileContent").value };
}

const up = await api("/documents", { body: uploadBody });
// up.document_id, up.bytes → then poll GET /status
```

#### Polling ingestion and OCR confirmation

Upload only writes to S3. Processing is async (S3 → SQS → document-processor). The UI polls `GET /status?document_id=…` every 1s (90s timeout) until status leaves `pending` / `processing`.

If status is `awaiting_ocr_confirmation` (PDF with no embedded text layer), the UI shows a Yes/No box and calls:

```javascript
await api("/documents-decision", {
  body: { document_id: documentId, decision }, // "ocr" | "cancel"
});
```

Choosing OCR starts a second poll round (also excluding the stale `awaiting_ocr_confirmation` row until the resumed run overwrites it). Cancel stops without Textract.

On success, the right pane replays the ingestion `trace` and shows parent/child chunk counts.

#### Chat — session, answer, tags

Each browser session gets a `session_id` (UUID). **Phiên mới** rotates it so multi-turn rewrite / history starts clean.

```javascript
const res = await api("/chat", {
  body: { question, session_id: state.sessionId },
});
await replayTrace(res.trace);
addMessage("bot", res.answer, res); // tags: cache hit, sources, …
```

Bot bubbles show metadata tags when present (`cached`, source document ids). HTTP errors with `retryable: true` (e.g. Bedrock throttle → 429) are labeled so the user knows to wait and retry.

#### Right pane — real timings, compressed animation

Animation is not fake:

| Flow | Timing source |
|---|---|
| Query | `POST /chat` response field `trace` (per-step ms) |
| Ingest | document-processor writes progress to DynamoDB; UI reads it via `GET /status` |

`replayTrace` plays steps in order with compression (`compress ≈ 0.35`, cap 1.2s per step) so a 7s generation step does not freeze the UI for 7s — the **displayed ms label stays the real server value**.

Steps the backend never ran (cache hit skips retrieval/generation, first-turn skip of query rewrite, etc.) are marked **skipped** (dashed/grey) with a reason, not highlighted falsely.

{{% notice tip %}}
Ask the **same** question again after **Phiên mới** to see a cache hit: usually one lit step, sub-second response, and a `cache hit` tag on the bot message.
{{% /notice %}}

---

Next: [5.7.3 - Deployment and Hosting](../5.7.3-Deployment-Hosting/)
