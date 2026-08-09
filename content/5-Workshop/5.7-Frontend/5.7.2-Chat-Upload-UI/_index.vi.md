---
title: "Giao diện Chat và Upload tài liệu"
date: 2026-08-07
weight: 2
chapter: false
pre: " <b> 5.7.2. </b> "
---

Trang này trình bày luồng upload và chat ở pane trái trong `ui/index.html`, cùng animation pipeline ở pane phải — phát lại thời gian thật từ backend.

#### Phiên hội thoại

`session_id` sinh phía client (`"ui-"` + chuỗi ngẫu nhiên / UUID), giữ trong **bộ nhớ** (không phải cookie/`localStorage`) — bấm **Phiên mới** sinh ID khác và xóa khung chat.

{{% notice note %}}
📌 `session_id` chính là khóa mà `chat_engine` dùng để tra lịch sử hội thoại (bảng `chat_history`), quyết định **có bỏ qua cache hay không**, và **có chạy query-rewriting hay không** (xem logic `cacheable = not history` ở [5.4.3](../../5.4-Realtime-QA/5.4.3-Cache-Lookup-Query-Rewriting/)). Bấm **Phiên mới** trên UI thực chất là cách thủ công để buộc lại trạng thái “chưa có lịch sử” ở backend.
{{% /notice %}}

#### Tag ngữ cảnh trên mỗi câu trả lời

Mỗi câu trả lời hiển thị kèm **tag ngữ cảnh** lấy trực tiếp từ response backend, **không suy diễn**:

| Tag | Điều kiện hiển thị |
|---|---|
| `cache hit` | Response có `cached: true` |
| đã viết lại câu hỏi | Response có trường `rewritten_query` |
| guardrail chặn | Response có `blocked: true` |
| 1 tag riêng cho mỗi tên file | Từng phần tử trong mảng `sources[]` |

```javascript
const res = await api("/chat", {
  body: { question, session_id: state.sessionId },
});
await replayTrace(res.trace);
addMessage("bot", res.answer, res); // tag từ cached / rewritten_query / sources / …
```

![Câu trả lời với các tag ngữ cảnh: cache hit, rewritten, sources](../images/03-answer-context-tags.png)
*Bubble bot kèm tag `cache hit` và tag tên file nguồn.*

#### Xử lý throttle (429)

Khi Bedrock quá tải, backend trả `429` kèm `{ retryable: true }` (xem [5.4.6](../../5.4-Realtime-QA/5.4.6-Error-Handling-OCR-Decision/)). Giao diện phân biệt lỗi chờ được với lỗi thật bằng icon **`⏳`** / **`⚠️`**:

```javascript
const retryable = err.data?.retryable;
addMessage("bot", (retryable ? "⏳ " : "⚠️ ") + err.message);
```

#### Tải tài liệu lên — rẽ nhánh theo phần mở rộng file

Rẽ nhánh khớp tập `OCR_EXTENSIONS`/PDF phía backend (xem [5.3.3](../../5.3-Data-Ingestion/5.3.3-Text-Extraction/)):

| Loại | Extension | Cách trình duyệt đọc | Body gửi `POST /documents` |
|---|---|---|---|
| Text | `.txt`, `.md`, `.csv`, `.json`, `.log`, `.htm`, `.html` | `file.text()` | `{ filename, content }` |
| Nhị phân (OCR) | `.pdf`, `.png`, `.jpg`, `.jpeg`, `.tiff`, `.tif` | `arrayBuffer()` → base64 | `{ filename, content_base64, content_type }` |

{{% notice warning %}}
Với file nhị phân, mã hóa base64 là **bắt buộc**. Đọc bằng `file.text()` sẽ **làm hỏng byte nhị phân** trước khi tới Textract (decode UTF-8 một file ảnh/PDF phá dữ liệu gốc). Với các file này, ô soạn thảo chuyển `readOnly`, chỉ hiện placeholder thông tin file thay vì nội dung.
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

Sau khi `POST /documents` thành công, giao diện **poll `GET /status` mỗi giây, tối đa 90 giây** để theo dõi tiến trình bất đồng bộ (S3 → SQS → Lambda), phát lại animation ở khu quan sát đúng theo `trace` mà `document-processor` ghi vào bảng `ingestion-status`.

![Animation pipeline chạy theo trace thật từ ingestion-status](../images/04-pipeline-animation-trace.png)
*Khu quan sát bên phải sáng dần từng bước theo đúng trace thời gian thật.*

#### Dialog xác nhận OCR (human-in-the-loop)

Khi `/status` trả `status: "awaiting_ocr_confirmation"` (PDF không có lớp text nhúng sẵn), giao diện:

1. **Dừng poll**.
2. Hiện hộp thoại Có/Không (dựng bằng `Promise` chờ người dùng bấm nút).
3. Gửi quyết định qua `POST /documents-decision`.
4. **Poll lại lần 2**.

{{% notice warning %}}
**Race condition cần lưu ý:** ở lần poll thứ 2 (sau khi gửi quyết định), giao diện **loại trừ luôn cả trạng thái `awaiting_ocr_confirmation` khỏi điều kiện dừng** — để tránh đọc trúng bản ghi cũ chưa kịp bị Lambda ghi đè. Nếu không loại trừ, poll ngay sau quyết định vẫn có thể thấy status cũ và hỏi lại người dùng lần nữa.
{{% /notice %}}

```javascript
async function pollStatus(documentId, { excludeAwaitingOcr = false } = {}) {
  for (let i = 0; i < 90; i++) {
    const result = await getStatus(documentId);
    const isTerminal = result.status === "completed" || result.status === "cancelled";
    const isAwaitingOcr = !excludeAwaitingOcr && result.status === "awaiting_ocr_confirmation";
    if (isTerminal || isAwaitingOcr) return result;
    await sleep(1000);
  }
}
```

![Dialog xác nhận OCR với 2 lựa chọn Có/Không](../images/05-ocr-confirm-dialog.png)
*Hộp thoại xác nhận OCR, dựng bằng Promise chờ người dùng bấm nút.*

#### Pane phải — thời gian thật, animation nén

| Luồng | Nguồn thời gian |
|---|---|
| Hỏi đáp | Trường `trace` trong response `POST /chat` (ms từng bước) |
| Nạp tài liệu | document-processor ghi tiến trình DynamoDB; UI đọc qua `GET /status` |

`replayTrace` chạy tuần tự với hệ số nén (`compress ≈ 0.35`, tối đa ~1.2s mỗi bước) để bước sinh câu trả lời 7s không làm UI đứng 7s — **nhãn ms vẫn là số thật phía server**. Bước backend không chạy (cache hit bỏ retrieval/generation, lượt đầu bỏ query rewrite, …) được đánh dấu **skipped** (xám, nét đứt) kèm lý do.

{{% notice tip %}}
Hỏi lại **y hệt** trong **cùng phiên** để thấy cache hit: thường chỉ sáng 1 bước, phản hồi dưới 1 giây, và tag `cache hit` trên tin nhắn bot. Dùng **Phiên mới** khi muốn demo multi-turn / rewrite từ trạng thái sạch.
{{% /notice %}}

---

Tiếp theo: [5.7.3 - Triển khai và Hosting](../5.7.3-Deployment-Hosting/)
