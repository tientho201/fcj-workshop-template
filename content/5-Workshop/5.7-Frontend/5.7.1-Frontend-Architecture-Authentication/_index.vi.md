---
title: "Kiến trúc Frontend và Authentication"
date: 2026-08-07
weight: 1
chapter: false
pre: " <b> 5.7.1. </b> "
---

Frontend nằm trong `ui/index.html` — một file HTML/CSS/JS tự chứa. Không React, không bundler, không `package.json`. Trang này trình bày cơ chế đăng nhập và cách trình duyệt gắn JWT Cognito trước khi gọi API Luồng 2.

#### Cơ chế đăng nhập

**Không dùng AWS Amplify hay SDK Cognito nào** — giao diện gọi thẳng API `InitiateAuth` của Cognito Identity Provider qua `fetch()` thuần:

```javascript
fetch(`https://cognito-idp.${region}.amazonaws.com/`, {
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
```

App Client phía Terraform là **public client** (`generate_secret = false`) — phù hợp vì code chạy hoàn toàn ở trình duyệt, không có nơi nào an toàn để giữ client secret.

![Gọi trực tiếp InitiateAuth qua fetch, xem trong DevTools Network](../images/01-initiate-auth-network-tab.png)
*Tab Network: POST tới `cognito-idp.<region>.amazonaws.com` với header `X-Amz-Target`.*

#### Vòng đời token — không có refresh flow

```javascript
// Thành công → lưu IdToken trong biến JS state, chỉ tồn tại trong bộ nhớ tab hiện tại
state.token = response.AuthenticationResult.IdToken;

// Gắn vào header Authorization của mọi request tới API Gateway
fetch(apiUrl, { headers: { Authorization: state.token } });
```

{{% notice warning %}}
**Đây là điểm đơn giản hóa có chủ đích, không phải thiếu sót:** giao diện **không triển khai refresh token flow**. `access_token_validity` / `id_token_validity` = 60 phút (cấu hình Terraform ở Luồng 2) — khi hết hạn, **người dùng phải đăng nhập lại thủ công**. Với bảng điều khiển nội bộ dùng để demo/test, đây là đánh đổi hợp lý giữa độ phức tạp code và trải nghiệm.
{{% /notice %}}

**`state.token` chỉ tồn tại trong bộ nhớ (biến JS), không lưu vào `localStorage`/cookie** — refresh trang là mất token, phải đăng nhập lại. Ngược lại, **cấu hình kết nối** (API URL, Client ID, Region, email) được lưu ở `localStorage` để lần sau khỏi gõ lại; **mật khẩu không bao giờ được lưu**, chỉ tồn tại trong ô input lúc gõ.

| Dữ liệu | Nơi lưu | Tồn tại tới khi nào |
|---|---|---|
| `IdToken` | Biến JS (`state.token`) | Đóng tab hoặc reload trang |
| API URL, Client ID, Region, email | `localStorage` (`rag.*`) | Người dùng tự xóa hoặc đổi giá trị mới |
| Mật khẩu | Không lưu | Chỉ trong lúc gõ vào ô input |

Mục **1 · Kết nối** điền từ Terraform output sau apply:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
```

{{% notice warning %}}
File example Terraform đặt `api_require_api_key = true`, nghĩa là mọi route cần header `x-api-key`. UI hiện **không có ô API key** và không gửi header đó. Đặt `api_require_api_key = false`, hoặc bổ sung key từ `terraform output -raw api_key_value` vào helper `api()` trước khi gọi backend.
{{% /notice %}}

#### API Gateway phía nhận

4 route (`/chat`, `/documents`, `/status`, `/documents-decision`) đều gắn `COGNITO_USER_POOLS` authorizer; chỉ preflight `OPTIONS` là `authorization = "NONE"` (bắt buộc — trình duyệt gửi preflight không kèm `Authorization`). `OPTIONS` chỉ trả header CORS qua tích hợp `MOCK`, không chạm Lambda hay dữ liệu thật (chi tiết hạ tầng: [5.4.1](../../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)).

Mọi request backend đi qua helper `api()` — ghép endpoint đã cấu hình và gắn `Authorization: state.token`. Nút Upload / Gửi câu hỏi chỉ mở sau khi login. Challenge phụ (MFA, …) báo lỗi rõ — MFA mặc định tắt trong stack này.

![Token hết hạn, giao diện báo lỗi rõ ràng yêu cầu đăng nhập lại](../images/02-token-expired-message.png)
*Sau ~60 phút, thao tác tiếp theo trả 401; UI yêu cầu đăng nhập lại thay vì treo im lặng.*

---

Tiếp theo: [5.7.2 - Giao diện Chat và Upload tài liệu](../5.7.2-Chat-Upload-UI/)
