---
title: "Triển khai và Hosting"
date: 2026-08-07
weight: 3
chapter: false
pre: " <b> 5.7.3. </b> "
---

Stack này **không** deploy frontend lên Amplify, S3 static website, hay CloudFront. UI là file local `ui/index.html`, mở bằng trình duyệt và gọi API Gateway + Cognito do Terraform tạo.

#### Mở giao diện

```bash
# từ thư mục gốc repo RAGonAWS
open ui/index.html          # macOS
# hoặc nháy đúp file / start ui/index.html trên Windows
```

Không bước build. Không cần server Node local cho luồng mặc định.

#### Điền cấu hình sau apply

Sau `terraform apply` (hoặc `scripts/up.sh`), lấy lại output — API Gateway và Cognito App Client đổi ID khi tạo lại stack:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
# chỉ khi api_require_api_key = true (UI mặc định chưa gửi):
# terraform output -raw api_key_value
```

Dán vào mục **1 · Kết nối** kèm region (ví dụ `ap-southeast-1`). Các trường trừ mật khẩu được lưu `localStorage` cho lần mở sau. Nếu bắt buộc API key: tắt trong `terraform.tfvars` hoặc bổ sung UI (xem [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)).

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

| Giới hạn | Chi tiết |
|---|---|
| Body API Gateway | Tối đa 10 MB; PDF/ảnh base64 phồng ~33% nên dung lượng nhị phân thực nhận được nhỏ hơn |
| PDF nhiều trang | Textract đồng bộ (`detect_document_text`) chỉ **1 trang** — nhiều trang báo lỗi rõ |
| Quota Bedrock | Gửi dồn có thể HTTP 429 (`retryable`) — đợi rồi thử lại |
| Hosting | Frontend không nằm trong vòng Terraform destroy/apply; chỉ mở file local |

{{% notice note %}}
Nếu sau này host file này trên S3/CloudFront, cần bổ sung CORS và cấu hình Cognito app client — **ngoài phạm vi** stack thực tập hiện tại.
{{% /notice %}}

---

Tiếp theo: [5.7.4 - Kiểm thử end-to-end](../5.7.4-Test-end-to-end/)
