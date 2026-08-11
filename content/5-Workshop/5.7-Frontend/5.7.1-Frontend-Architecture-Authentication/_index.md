---
title: "Frontend Architecture and Authentication"
date: 2026-08-07
weight: 1
chapter: false
pre: " <b> 5.7.1. </b> "
---

The frontend lives in `ui/index.html` — one self-contained HTML/CSS/JS file. There is no React, no bundler, and no `package.json`. This page covers login and how the browser attaches a Cognito JWT before calling the Stream 2 API.

#### Login mechanism

**No AWS Amplify and no Cognito SDK** — the UI calls Cognito Identity Provider `InitiateAuth` directly with plain `fetch()`:

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

The Terraform App Client is a **public client** (`generate_secret = false`) — correct for browser-only code, where there is nowhere safe to keep a client secret.

#### Token lifecycle — no refresh flow

```javascript
// Success → keep IdToken in JS state (tab memory only)
state.token = response.AuthenticationResult.IdToken;

// Attach to Authorization on every API Gateway call
fetch(apiUrl, { headers: { Authorization: state.token } });
```

{{% notice warning %}}
**Intentional simplification, not a missing feature:** the UI **does not implement a refresh-token flow**. `access_token_validity` / `id_token_validity` = 60 minutes (Terraform in Stream 2). When the token expires, the user **must sign in again**. For an internal demo/test console, that trade-off keeps the code small.
{{% /notice %}}

`state.token` lives only in memory (a JS variable) — **not** in `localStorage` or cookies. Reload the page and the token is gone; sign in again. Connection settings (API URL, Client ID, region, email) **are** stored in `localStorage` so you do not retype them; the **password is never stored**, only present in the input while typing.

| Data                              | Stored where                | Lives until                       |
| --------------------------------- | --------------------------- | --------------------------------- |
| `IdToken`                         | JS variable (`state.token`) | Tab close or page reload          |
| API URL, Client ID, Region, email | `localStorage` (`rag.*`)    | User clears storage or overwrites |
| Password                          | Not stored                  | Only while typing in the input    |

Section **1 · Kết nối** is filled from Terraform outputs after apply:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
```

{{% notice warning %}}
Terraform example sets `api_require_api_key = true`, which requires header `x-api-key` on every route. The current UI has **no API-key field** and does not send that header. Use `api_require_api_key = false`, or add the key from `terraform output -raw api_key_value` to the `api()` helper before calling the backend.
{{% /notice %}}

#### API Gateway on the receiving side

All four routes (`/chat`, `/documents`, `/status`, `/documents-decision`) use a `COGNITO_USER_POOLS` authorizer. Only preflight `OPTIONS` is `authorization = "NONE"` (required — the browser does not send `Authorization` on preflight). `OPTIONS` returns CORS headers via a `MOCK` integration and never hits Lambda or real data (infra detail: [5.4.1](../../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)).

Every backend call goes through a shared `api()` helper that prefixes the configured endpoint and sets `Authorization: state.token`. Upload and Ask stay disabled until login succeeds. Extra Cognito challenges (MFA, etc.) surface as errors — MFA is off by default in this stack.

---

#### Next content

- [5.7.2 - Chat Interface and Document Upload](../5.7.2-Chat-Upload-UI/)
