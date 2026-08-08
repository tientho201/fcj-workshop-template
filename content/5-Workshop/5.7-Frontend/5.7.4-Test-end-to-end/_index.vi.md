---
title: "Kiểm thử end-to-end"
date: 2026-08-07
weight: 4
chapter: false
pre: " <b> 5.7.4. </b> "
---

{{% notice note %}}
📌 Trang này dựng **khung kịch bản test** bám danh sách “những thứ nên thử” trong `ui/README.md`. Số liệu thời gian thật và ảnh chụp màn hình cần tự chạy và điền khi viết báo cáo.
{{% /notice %}}

Sau [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)–[5.7.3](../5.7.3-Deployment-Hosting/), mở `ui/index.html`, dán cấu hình từ `terraform output`, đăng nhập user Cognito.

#### Kịch bản kiểm thử giao diện

| # | Kịch bản | Kỳ vọng |
|---|---|---|
| 1 | Đăng nhập với tài khoản Cognito hợp lệ | Nhận `IdToken`; chấm trạng thái chuyển xanh; Upload / Gửi bật |
| 2 | Đăng nhập sai mật khẩu | Hiện lỗi rõ ràng, không crash giao diện; nút vẫn tắt |
| 3 | Tải tài liệu `.txt`/`.md` (text) | Hiện được nội dung trong ô soạn thảo, sửa tay được trước khi gửi |
| 4 | Tải PDF có lớp text sẵn | Ingest xong không qua bước xác nhận OCR, trích xuất bằng `pypdf` |
| 5 | Tải PDF dạng scan (không có text) — chọn **Có** | Hiện dialog xác nhận OCR → chạy Textract → tiếp tục pipeline |
| 6 | Tải PDF dạng scan — chọn **Không** | Dừng ở trạng thái hủy, không tốn phí Textract |
| 7 | Hỏi về nội dung tài liệu vừa tải | Trả lời đúng, có tag tên file nguồn khi có |
| 8 | Hỏi lại y hệt câu vừa hỏi (**cùng phiên**) | Tag `cache hit`; chỉ 1 bước sáng ở khu quan sát; phản hồi &lt; 1s |
| 9 | Hỏi câu mơ hồ (“còn cái kia thì sao?”) trong **cùng phiên** | Bước rewrite sáng lên; log in câu đã viết lại |
| 10 | Hỏi điều không có trong tài liệu | Model từ chối / không bịa tự do |
| 11 | Gửi nhiều câu liên tiếp thật nhanh | Có thể gặp 429, hiện icon `⏳` khác với lỗi thường; thử lại sau vài giây được |

![Bảng kết quả test 11 kịch bản với ảnh chụp thật](../images/07-test-scenarios-real-run.png)
*(Thay bằng ảnh chụp màn hình thật khi bạn chạy đủ 11 kịch bản.)*

{{% notice tip %}}
Nếu ingest quá ~90s, xem CloudWatch Logs `document-processor` và độ sâu SQS/DLQ (alarm Luồng 3). Timeout UI cố ý để pipeline kẹt lộ ra thay vì quay mãi.
{{% /notice %}}

#### Checklist trước khi coi là “đạt”

- [ ] Token hết hạn (đợi &gt; 60 phút) → thao tác tiếp báo lỗi rõ ràng, không treo im lặng
- [ ] Upload file nhị phân &gt; ~7 MB → xác nhận nhận được thông báo giới hạn body 10 MB của API Gateway
- [ ] Reload trang giữa lúc đang poll `/status` → không crash; cấu hình vẫn còn nhờ `localStorage`
- [ ] Responsive: thu nhỏ dưới 900px → layout chuyển 1 cột
- [ ] Đã dán `terraform output` vào **1 · Kết nối**; mật khẩu không nằm trong `localStorage`
- [ ] Rõ frontend chỉ chạy local (không Amplify / S3 website trong stack này)

{{% notice tip %}}
Kịch bản #11 (429) và checklist giới hạn body 10 MB dễ bị bỏ sót khi demo qua loa — đây là 2 giới hạn thực tế đáng ghi vào phần “Kết quả kiểm thử”.
{{% /notice %}}

#### Kết quả đạt được

- UI một file gọi đủ bốn route Luồng 2 với JWT Cognito thật.
- Upload phân biệt text và nhị phân (base64) cho Textract.
- Animation pipeline phản ánh `trace` / status server, kể cả bước skipped.
- Cache hit và rewrite multi-turn nhìn thấy được mà không cần mở AWS Console.
