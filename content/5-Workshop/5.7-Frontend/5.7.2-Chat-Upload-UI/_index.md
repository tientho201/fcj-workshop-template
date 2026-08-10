---
title: "Chat Interface and Document Upload"
date: 2026-08-07
weight: 2
chapter: false
pre: " <b> 5.7.2. </b> "
---

This page covers the left-pane upload and chat flows in `ui/index.html`, plus the right-pane pipeline animation that replays real backend timings.

#### Conversation session

`session_id` is created on the client (`"ui-"` + random string / UUID) and kept in **memory** only (not cookie / `localStorage`). **Phiên mới** rotates the ID and clears the chat pane.

{{% notice note %}}
📌 `session_id` is the key `chat_engine` uses to load history (`chat_history`), decide **whether semantic/exact cache is eligible**, and **whether to run query rewriting** (see `cacheable = not history` in [5.4.3](../../5.4-Realtime-QA/5.4.3-Cache-Lookup-Query-Rewriting/)). Clicking **Phiên mới** is the manual way to force a “no history yet” state on the backend.
{{% /notice %}}

#### Context tags on each answer

Each bot answer shows **context tags** taken straight from the backend response — no client-side guessing:

| Tag                     | Shown when                          |
| ----------------------- | ----------------------------------- |
| `cache hit`             | Response has `cached: true`         |
| rewritten query         | Response includes `rewritten_query` |
| guardrail blocked       | Response has `blocked: true`        |
| one tag per source file | Each entry in `sources[]`           |

```javascript
const res = await api("/chat", {
  body: { question, session_id: state.sessionId },
});
await replayTrace(res.trace);
addMessage("bot", res.answer, res); // tags from cached / rewritten_query / sources / …
```

#### Throttle handling (429)

When Bedrock is overloaded, the backend returns `429` with `{ retryable: true }` (see [5.4.6](../../5.4-Realtime-QA/5.4.6-Error-Handling-OCR-Decision/)). The UI distinguishes retryable failures from hard errors (`⏳` vs `⚠️`):

```javascript
const retryable = err.data?.retryable;
addMessage("bot", (retryable ? "⏳ " : "⚠️ ") + err.message);
```

#### Document upload — branch by file extension

Branching matches the backend `OCR_EXTENSIONS` / PDF set (see [5.3.3](../../5.3-Data-Ingestion/5.3.3-Text-Extraction/)):

| Kind         | Extensions                                              | How the browser reads    | Body sent to `POST /documents`               |
| ------------ | ------------------------------------------------------- | ------------------------ | -------------------------------------------- |
| Text         | `.txt`, `.md`, `.csv`, `.json`, `.log`, `.htm`, `.html` | `file.text()`            | `{ filename, content }`                      |
| Binary (OCR) | `.pdf`, `.png`, `.jpg`, `.jpeg`, `.tiff`, `.tif`        | `arrayBuffer()` → base64 | `{ filename, content_base64, content_type }` |

{{% notice warning %}}
For binary files, base64 encoding is **required**. Reading with `file.text()` **corrupts binary bytes** before they reach Textract (UTF-8 decode of an image/PDF destroys the original data). For those files the editor becomes `readOnly` and shows a placeholder instead of content.
{{% /notice %}}

```javascript
async function readFileForUpload(file) {
  const ext = file.name.split(".").pop().toLowerCase();
  if (TEXT_EXTENSIONS.includes(ext)) {
    return { content: await file.text(), content_base64: null };
  }
  const buffer = await file.arrayBuffer();
  return { content: null, content_base64: arrayBufferToBase64(buffer) };
}
```

After a successful `POST /documents`, the UI **polls `GET /status` every second for up to 90 seconds** to follow the async path (S3 → SQS → Lambda) and replay the right-pane animation from the `trace` that `document-processor` writes into `ingestion-status`.

#### OCR confirmation dialog (human-in-the-loop)

When `/status` returns `status: "awaiting_ocr_confirmation"` (PDF with no embedded text layer), the UI:

1. **Stops polling**.
2. Shows a Yes/No box (built as a `Promise` that waits for the button click).
3. Sends the decision via `POST /documents-decision`.
4. **Polls a second time**.

{{% notice warning %}}
**Race to watch:** on the second poll (after sending the decision), the UI **also excludes `awaiting_ocr_confirmation` from the stop condition** — so it does not re-read a stale row before Lambda overwrites it. Without that exclusion, the first poll right after the decision can still see the old `awaiting_ocr_confirmation` and ask the user again.
{{% /notice %}}

```javascript
async function pollStatus(documentId, { excludeAwaitingOcr = false } = {}) {
  for (let i = 0; i < 90; i++) {
    const result = await getStatus(documentId);
    const isTerminal =
      result.status === "completed" || result.status === "cancelled";
    const isAwaitingOcr =
      !excludeAwaitingOcr && result.status === "awaiting_ocr_confirmation";
    if (isTerminal || isAwaitingOcr) return result;
    await sleep(1000);
  }
}
```

![OCR confirmation dialog with Yes/No](../images/05-ocr-confirm-dialog.png)
_OCR confirm box built as a Promise waiting for the user click._

#### Right pane — real timings, compressed animation

| Flow   | Timing source                                                                 |
| ------ | ----------------------------------------------------------------------------- |
| Query  | `POST /chat` response field `trace` (per-step ms)                             |
| Ingest | document-processor writes progress to DynamoDB; UI reads it via `GET /status` |

`replayTrace` plays steps in order with compression (`compress ≈ 0.35`, cap ~1.2s per step) so a 7s generation step does not freeze the UI for 7s — the **displayed ms label stays the real server value**. Steps the backend never ran (cache hit skips retrieval/generation, first-turn skip of query rewrite, etc.) are marked **skipped** (dashed/grey) with a reason.

{{% notice tip %}}
Ask the **same** question again in the **same** session to see a cache hit: usually one lit step, sub-second response, and a `cache hit` tag on the bot message. Use **Phiên mới** when you want a clean multi-turn / rewrite demo instead.
{{% /notice %}}

---

#### Next content

- [5.7.3 - Deployment and Hosting](../5.7.3-Deployment-Hosting/)
