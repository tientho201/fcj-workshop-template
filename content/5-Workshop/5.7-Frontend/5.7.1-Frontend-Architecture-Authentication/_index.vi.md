---
title: "Kiến trúc Frontend và Authentication"
date: 2026-08-07
weight: 1
chapter: false
pre: " <b> 5.7.1. </b> "
---

Frontend nằm trong `ui/index.html` — một file HTML/CSS/JS tự chứa. Không React, không bundler, không `package.json`. Trang này trình bày bố cục và cách trình duyệt đăng nhập Cognito trước khi gọi API Luồng 2.

#### Bố cục

Giao diện chia 2 pane:

| Pane | Vai trò |
|---|---|
| Trái | Cấu hình kết nối, upload tài liệu, chat |
| Phải | Animation từng bước pipeline + nhật ký chi tiết |

Màn hẹp (`max-width: 900px`) xếp thành một cột.

#### Cấu hình kết nối

Mục **1 · Kết nối** lấy giá trị từ Terraform output:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
```

| Trường | Nguồn |
|---|---|
| API endpoint URL | `api_gateway_endpoint_url` |
| Cognito App Client ID | `cognito_app_client_id` |
| Region | ví dụ `ap-southeast-1` |
| Email / mật khẩu | user Cognito |

`apiUrl`, `clientId`, `region`, `username` được lưu trong `localStorage` (khóa `rag.*`). Mật khẩu **không** lưu — ô password được xóa sau khi đăng nhập thành công.

{{% notice warning %}}
File example Terraform đặt `api_require_api_key = true`, nghĩa là mọi route cần header `x-api-key`. UI hiện **không có ô API key** và không gửi header đó. Đặt `api_require_api_key = false`, hoặc bổ sung key từ `terraform output -raw api_key_value` vào `api()` trước khi gọi backend.
{{% /notice %}}

#### Đăng nhập Cognito (browser → Cognito API)

Đăng nhập gọi thẳng Cognito Identity Provider từ trình duyệt — không Hosted UI, không Amplify SDK:

```javascript
async function cognitoLogin(region, clientId, username, password) {
  const res = await fetch(`https://cognito-idp.${region}.amazonaws.com/`, {
    method: "POST",
    headers: {
      "Content-Type": "application/x-amz-json-1.1",
      "X-Amz-Target": "AWSCognitoIdentityProviderService.InitiateAuth",
    },
    body: JSON.stringify({
      AuthFlow: "USER_PASSWORD_AUTH",
      ClientId: clientId,
      AuthParameters: { USERNAME: username, PASSWORD: password },
    }),
  });
  const data = await res.json();
  if (!data.AuthenticationResult?.IdToken) {
    throw new Error(`Cognito yêu cầu bước bổ sung: ${data.ChallengeName || "không rõ"}`);
  }
  return data.AuthenticationResult.IdToken;
}
```

Thành công thì UI giữ **ID token** trong bộ nhớ (`state.token`) và bật đèn auth xanh. Nút Upload / Gửi câu hỏi chỉ mở sau khi login. Challenge phụ (MFA, …) báo lỗi rõ — MFA mặc định tắt trong stack này (xem cấu hình Cognito Luồng 2).

#### Helper gọi API đã xác thực

Mọi request backend đi qua `api()` — ghép endpoint đã cấu hình và gắn JWT Cognito:

```javascript
async function api(path, { method = "POST", body, query } = {}) {
  const base = $("apiUrl").value.trim().replace(/\/+$/, "");
  let url = `${base}${path}`;
  if (query) url += "?" + new URLSearchParams(query).toString();

  const res = await fetch(url, {
    method,
    headers: {
      "Content-Type": "application/json",
      Authorization: state.token, // Cognito ID token (API Gateway Cognito authorizer)
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  // …parse JSON, throw khi !res.ok
  return data;
}
```

Bốn route dùng sau (`/chat`, `/documents`, `/status`, `/documents-decision`) đều dùng helper này. Hạ tầng API Gateway / Cognito: [5.4.1](../../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/).

---

Tiếp theo: [5.7.2 - Giao diện Chat và Upload tài liệu](../5.7.2-Chat-Upload-UI/)
