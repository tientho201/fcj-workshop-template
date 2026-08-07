---
title: "Giao diện Chat và Upload tài liệu"
date: 2026-08-07
weight: 2
chapter: false
pre: " <b> 5.7.2. </b> "
---

Trang này trình bày luồng upload và chat ở pane trái trong `ui/index.html`, cùng animation pipeline ở pane phải — phát lại thời gian thật từ backend.

#### Upload tài liệu — text và nhị phân

| Loại | Extension | Cách trình duyệt đọc | Body gửi `POST /documents` |
|---|---|---|---|
| Text | `.txt`, `.md`, `.csv`, `.json`, `.log`, `.htm`, `.html` | `file.text()` | `{ filename, content }` |
| Nhị phân (OCR) | `.pdf`, `.png`, `.jpg`, `.jpeg`, `.tiff`, `.tif` | `arrayBuffer()` → base64 | `{ filename, content_base64, content_type }` |

File nhị phân **không** dùng `file.text()` — sẽ làm hỏng byte trước khi tới Textract. Ô soạn thảo chuyển read-only và hiện placeholder; muốn đổi nội dung thì chọn lại file.

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
// up.document_id, up.bytes → sau đó poll GET /status
```

#### Poll ingestion và xác nhận OCR

Upload chỉ ghi S3. Xử lý bất đồng bộ (S3 → SQS → document-processor). UI poll `GET /status?document_id=…` mỗi 1s (timeout 90s) đến khi status thoát `pending` / `processing`.

Nếu status là `awaiting_ocr_confirmation` (PDF không có lớp text nhúng), UI hiện hộp Có/Không rồi gọi:

```javascript
await api("/documents-decision", {
  body: { document_id: documentId, decision }, // "ocr" | "cancel"
});
```

Chọn OCR → poll vòng hai (loại cả hàng `awaiting_ocr_confirmation` cũ cho tới khi run resume ghi đè). Huỷ thì dừng, không chạy Textract.

Thành công thì pane phải phát lại `trace` ingestion và hiện số parent/child chunk.

#### Chat — phiên, câu trả lời, tag

Mỗi lần mở UI có `session_id` (UUID). **Phiên mới** đổi id để rewrite / history multi-turn bắt đầu sạch.

```javascript
const res = await api("/chat", {
  body: { question, session_id: state.sessionId },
});
await replayTrace(res.trace);
addMessage("bot", res.answer, res); // tag: cache hit, sources, …
```

Bubble bot hiện tag metadata khi có (`cached`, id tài liệu nguồn). Lỗi HTTP với `retryable: true` (ví dụ Bedrock throttle → 429) được gắn nhãn để người dùng biết đợi rồi thử lại.

#### Pane phải — thời gian thật, animation nén

Animation không phải hiệu ứng giả:

| Luồng | Nguồn thời gian |
|---|---|
| Hỏi đáp | Trường `trace` trong response `POST /chat` (ms từng bước) |
| Nạp tài liệu | document-processor ghi tiến trình DynamoDB; UI đọc qua `GET /status` |

`replayTrace` chạy tuần tự với hệ số nén (`compress ≈ 0.35`, tối đa 1.2s mỗi bước) để bước sinh câu trả lời 7s không làm UI đứng 7s — **nhãn ms vẫn là số thật phía server**.

Bước backend không chạy (cache hit bỏ retrieval/generation, lượt đầu bỏ query rewrite, …) được đánh dấu **skipped** (xám, nét đứt) kèm lý do — không tô sáng giả.

{{% notice tip %}}
Hỏi lại **y hệt** sau khi bấm **Phiên mới** để thấy cache hit: thường chỉ sáng 1 bước, phản hồi dưới 1 giây, và tag `cache hit` trên tin nhắn bot.
{{% /notice %}}

---

Tiếp theo: [5.7.3 - Triển khai và Hosting](../5.7.3-Deployment-Hosting/)
