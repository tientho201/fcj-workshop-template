---
title: "Frontend Architecture and Authentication"
date: 2026-08-07
weight: 1
chapter: false
pre: " <b> 5.7.1. </b> "
---

The frontend lives in `ui/index.html` — one self-contained HTML/CSS/JS file. There is no React, no bundler, and no `package.json`. This page covers layout and how the browser authenticates with Cognito before calling the Stream 2 API.

#### Layout

The UI is a two-pane grid:

| Pane | Role |
|---|---|
| Left | Connection config, document upload, chat |
| Right | Pipeline step animation + detailed logs |

On narrow screens (`max-width: 900px`) the grid collapses to one column.

#### Connection config

Section **1 · Kết nối** collects values from Terraform outputs:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
```

| Field | Source |
|---|---|
| API endpoint URL | `api_gateway_endpoint_url` |
| Cognito App Client ID | `cognito_app_client_id` |
| Region | e.g. `ap-southeast-1` |
| Email / password | Cognito user |

`apiUrl`, `clientId`, `region`, and `username` are saved under `localStorage` keys `rag.*`. The password is never persisted — the field is cleared after a successful login.

{{% notice warning %}}
Terraform example sets `api_require_api_key = true`, which requires header `x-api-key` on every route. The current UI has **no API-key field** and does not send that header. Use `api_require_api_key = false`, or add the key from `terraform output -raw api_key_value` to `api()` before calling the backend.
{{% /notice %}}

#### Cognito login (browser → Cognito API)

Login calls Cognito Identity Provider directly from the browser — no Hosted UI and no Amplify SDK:

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

On success the UI keeps the **ID token** in memory (`state.token`) and turns the auth indicator green. Upload and Ask stay disabled until login succeeds. Extra Cognito challenges (MFA, etc.) surface as errors — MFA is off by default in this stack (see Stream 2 Cognito settings).

#### Authenticated API helper

Every backend call goes through `api()`, which prefixes the configured endpoint and attaches the Cognito JWT:

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
  // …parse JSON, throw on !res.ok
  return data;
}
```

The four routes used later (`/chat`, `/documents`, `/status`, `/documents-decision`) all share this helper. API Gateway / Cognito infrastructure is in [5.4.1](../../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/).

---

Next: [5.7.2 - Chat Interface and Document Upload](../5.7.2-Chat-Upload-UI/)
