---
title: "Thử nghiệm hệ thống"
date: 2024-01-01
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---

#### Kịch bản kiểm thử Backend

| #   | Kịch bản                                                | Kỳ vọng                                                                                                                   |
| --- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| 1   | `terraform fmt -check` + `terraform validate`           | Pass — chạy tự động trong CI (`ci.yml`) trên mọi PR                                                                       |
| 2   | `ruff check modules`                                    | Pass — lint code Lambda, loại trừ `pypdf` đã vendor                                                                       |
| 3   | Ingest file text thuần                                  | Trích xuất trực tiếp, không qua Textract                                                                                  |
| 4   | Ingest PDF có lớp text                                  | pypdf đọc được, không tốn phí Textract                                                                                    |
| 5   | Ingest PDF dạng scan                                    | Dừng ở `awaiting_ocr_confirmation`, xác nhận Có → Textract OCR chạy đúng, xác nhận Không → trạng thái `cancelled`         |
| 6   | Re-upload cùng 1 file                                   | Chunk cũ bị xóa trước khi index lại, không nhân đôi dữ liệu                                                               |
| 7   | Hỏi câu có nghĩa tương đương nhưng khác từ với tài liệu | Cosine bắt được (nhánh vector), kể cả khi BM25 không match từ nào                                                         |
| 8   | Hỏi đúng 1 mã/tên riêng xuất hiện trong tài liệu        | BM25 kéo đúng chunk chứa từ đó lên top, dù cosine có thể xếp thấp hơn                                                     |
| 9   | Gọi `/chat` 2 lần với câu hỏi y hệt, khác session       | Request 2 **không** hit cache của request 1 nếu request 1's session đã có lịch sử (cache bị bỏ qua khi có history)        |
| 10  | Rút VPC endpoint `lambda` rồi gọi `/documents-decision` | Tái hiện đúng lỗi 504 đã từng gặp thật trong quá trình phát triển — xác nhận endpoint là nguyên nhân, không phải lỗi code |
| 11  | Bắn liên tiếp nhiều câu hỏi cùng lúc                    | Bedrock trả `ThrottlingException`, backend trả `429 {retryable:true}`, không crash Lambda                                 |

![Kết quả chạy CI (fmt/validate/lint) và test thủ công các kịch bản trên](../images/05-backend-test-results.png)
_Ảnh minh họa: kết quả `terraform validate`, `ruff check`, và log test kịch bản 7-8 (cosine vs BM25) trên CloudWatch._

#### Đã kiểm thử thật trong quá trình phát triển (không phải giả định)

{{% notice note %}}
📌 Mục **5, 6 và 10** ở trên là các lỗi/kịch bản **thực sự đã xảy ra và được sửa** trong lịch sử phát triển dự án

- **Bug thiếu VPC endpoint** cho Lambda control-plane API gây lỗi 504 khi `chat_engine` gọi thẳng `document_processor` — phát hiện qua test E2E thật, sửa bằng cách thêm `aws_vpc_endpoint "lambda"` (`modules/networking/main.tf`).
- **Luồng xác nhận OCR** (mục 5) đã test cả 2 nhánh (đồng ý/từ chối) bằng file PDF ảnh dựng tay (không có lớp text) và xác nhận đúng hành vi tạm dừng — không âm thầm chạy OCR hay âm thầm bỏ qua tài liệu.
  {{% /notice %}}

#### Chưa có test tự động (gap cần nêu rõ trong báo cáo)

{{% notice note %}}
📌 **Cập nhật:** xem [Kiểm thử hệ thống, trang 5.10.3](../../5.10-Testing/5.10.3-Khoang-trong-va-De-xuat/): 33 unit test (`pytest`) đã viết cho đúng các module nêu trên (`chunking.py`, `bm25.py`, `retrieval.py`, `vector_store.py`), chạy tự động trong CI. Phần **còn thiếu**: `handler.py` của cả 2 Lambda (code gọi trực tiếp `boto3`/AWS) vẫn chưa có test tự động.
{{% /notice %}}

#### Nội dung chi tiết

1. [Tầng 1 — Kiểm tra tĩnh tự động (CI)](5.10.1-Layer-1-Automated-Static-Analysis/)
2. [Tầng 2 — Kiểm thử thủ công/E2E](5.10.2-Layer-2-Manual-E2E-Testing/)
3. [Tầng 3 — Unit test tự động cho logic thuần](5.10.3-Layer-3-Automated-Unit-Testing/)
