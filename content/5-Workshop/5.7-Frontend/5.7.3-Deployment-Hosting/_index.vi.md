---
title: "Triển khai và Hosting"
date: 2026-08-07
weight: 3
chapter: false
pre: " <b> 5.7.3. </b> "
---

#### Không có hạ tầng hosting

{{% notice warning %}}
📌 Khác với thiết kế S3 + CloudFront + OAC thường dùng cho SPA, dự án này **không provision hạ tầng hosting nào cho UI** — không có trong bất kỳ file `.tf` nào (không có `cloudfront` / OAC cho console).
{{% /notice %}}

Vì `ui/index.html` là một file tĩnh tự chứa, không build step, cách “triển khai” đơn giản nhất — **và cũng là cách đang dùng** — là mở trực tiếp file bằng trình duyệt:

```bash
# từ thư mục gốc repo RAGonAWS
open ui/index.html          # macOS
# hoặc nháy đúp file / start ui/index.html trên Windows
```

Không cần server Node local, không cần domain, không cần SPA routing (chỉ 1 trang, không có route nào để 404).

#### Cấu hình lúc chạy (runtime config)

Vì không có build step, cấu hình môi trường (API endpoint, Cognito Client ID, region) **không được bake vào lúc build**. Người dùng tự điền vào form ở panel **1 · Kết nối** mỗi khi các giá trị đó đổi (API Gateway và Cognito App Client được cấp ID mới sau mỗi lần recreate). Giá trị được lưu `localStorage` để không phải nhập lại:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
# chỉ khi api_require_api_key = true (UI mặc định chưa gửi):
# terraform output -raw api_key_value
```

{{% notice tip %}}
API Gateway / Cognito Client ID đổi mỗi lần `terraform apply` lại từ đầu (destroy + tạo mới, không phải update tại chỗ). **Luôn chạy lại 2 lệnh `terraform output` ở trên sau mỗi lần apply lại toàn bộ hạ tầng**, rồi cập nhật panel Kết nối — nếu quên, giao diện sẽ gọi endpoint cũ đã không còn tồn tại.
{{% /notice %}}

Nếu bắt buộc API key: đặt `api_require_api_key = false` trong `terraform.tfvars` hoặc bổ sung UI (xem [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)).

#### Tạo user Cognito để test

```bash
aws cognito-idp admin-create-user \
  --user-pool-id <POOL_ID> \
  --username <email> \
  --user-attributes Name=email,Value=<email> Name=email_verified,Value=true \
  --message-action SUPPRESS

aws cognito-idp admin-set-user-password \
  --user-pool-id <POOL_ID> \
  --username <email> \
  --password '<Password>' \
  --permanent
```

`POOL_ID` lấy từ Terraform output / Cognito Console. Tài khoản demo mẫu ghi trong `ui/README.md` — có thể khác theo môi trường.

#### Giới hạn đã biết (UI + API)

| Giới hạn         | Chi tiết                                                                               |
| ---------------- | -------------------------------------------------------------------------------------- |
| Body API Gateway | Tối đa 10 MB; PDF/ảnh base64 phồng ~33% nên dung lượng nhị phân thực nhận được nhỏ hơn |
| PDF nhiều trang  | Textract đồng bộ (`detect_document_text`) chỉ **1 trang** — nhiều trang báo lỗi rõ     |
| Quota Bedrock    | Gửi dồn có thể HTTP 429 (`retryable`) — đợi rồi thử lại                                |
| Hosting          | Frontend không nằm trong vòng Terraform destroy/apply; chỉ mở file local               |
| Layout           | Dưới ~900px lưới xếp thành một cột                                                     |

#### Hướng mở rộng (chưa làm)

Nếu cần chia sẻ UI cho nhiều người không có quyền chạy `terraform output`, hướng hợp lý là thêm **1 bucket S3** (static website hoặc origin cho CloudFront + OAC) chỉ để host đúng 1 file HTML này — không cần build pipeline vì file đã tĩnh sẵn.

{{% notice note %}}
Việc này **chưa được triển khai** trong phạm vi stack thực tập hiện tại. Khi viết báo cáo nên trình bày đây là hướng phát triển tiếp theo — không phải phần đã hoàn thành. Host sau này cũng cần bổ sung CORS và cấu hình Cognito app client.
{{% /notice %}}

---

#### Nội dung tiếp theo

- [5.7.4 - Kiểm thử end-to-end](../5.7.4-Test-end-to-end/)
