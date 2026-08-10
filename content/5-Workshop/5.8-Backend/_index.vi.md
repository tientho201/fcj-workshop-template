---
title: "Backend"
date: 2024-01-01
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

#### Kiến trúc code Backend

Phần này mô tả các quyết định thiết kế **ở mức code**, khác với chương "Luồng xử lý" (5.3 → 5.6 — mô tả theo pipeline hạ tầng end-to-end). Cả 2 Lambda chính (`document_processor`, `chat_engine`) tuân theo một nguyên tắc xuyên suốt:

{{% notice note %}}
📌 **Zero external dependency** — ngoại trừ đúng 1 thư viện (`pypdf`, được vendor thẳng vào git thay vì cài lúc `terraform apply`), toàn bộ logic chỉ dùng `boto3`/`botocore` và thư viện chuẩn Python. Ngay cả client Redis cũng là một RESP client ~70 dòng **tự viết**, thay vì dùng `redis-py`.
{{% /notice %}}

{{% notice warning %}}
📌 **Cập nhật (đã đối chiếu lại với code thật):** có **4 file** dùng chung giữa 2 Lambda, không phải 2 như ghi nhận trước đây — `bm25.py`, `vector_store.py`, **`tracing.py`**, **`embeddings.py`**. Cả 4 tồn tại dưới dạng **bản sao y hệt** ở cả 2 Lambda — không dùng Lambda Layer hay package chung, giữ mỗi Lambda **tự chứa hoàn toàn**, đánh đổi lấy việc phải đồng bộ tay khi sửa. Rủi ro này **đã có test tự động kiểm tra** (`test_shared_copies_in_sync.py`, xem [5.8.4](5.8.4-Backend-Testing/)).
{{% /notice %}}

---

#### Nội dung chi tiết

1. [Thuật toán Retrieval](5.8.1-Retrieval-Algorithms/)
2. [Semantic Cache & Observability](5.8.2-Cache-va-Observability/)
3. [Xử lý lỗi & Bảo mật](5.8.3-Error-Handling-and-Security/)
4. [Kiểm thử Backend](5.8.4-Backend-Testing/)
