---
title: "Deployment and Hosting"
date: 2026-08-07
weight: 3
chapter: false
pre: " <b> 5.7.3. </b> "
---

This stack does **not** deploy the frontend to Amplify, S3 static website hosting, or CloudFront. The UI is a local file under `ui/index.html`, opened in a browser against the API Gateway + Cognito resources created by Terraform.

#### Open the UI

```bash
# from the RAGonAWS repo root
open ui/index.html          # macOS
# or double-click the file / start ui/index.html on Windows
```

No build step. No local Node server required for the default flow.

#### Wire config after apply

After `terraform apply` (or `scripts/up.sh`), refresh outputs — API Gateway and Cognito App Client IDs change when the stack is recreated:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
# only if api_require_api_key = true (UI does not send this yet by default):
# terraform output -raw api_key_value
```

Paste into section **1 · Kết nối** together with the region (e.g. `ap-southeast-1`). Non-password fields persist in `localStorage` for the next open. If API key is required, either disable it in `terraform.tfvars` or extend the UI (see [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)).

#### Create a Cognito test user

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

`POOL_ID` comes from Terraform outputs / the Cognito console. Example credentials used in demos are documented in `ui/README.md` and may differ per environment.

#### Known limits (UI + API)

| Limit | Detail |
|---|---|
| API Gateway body | 10 MB max; base64 PDF/images inflate ~33%, so usable binary size is smaller |
| Multi-page PDF | Sync Textract (`detect_document_text`) handles **1 page** only — multi-page fails with a clear error |
| Bedrock quota | Bursting requests can return HTTP 429 (`retryable`) — wait and retry |
| Hosting | Frontend is not part of the Terraform destroy/apply surface; only open the file locally |

{{% notice note %}}
If the team later hosts this file on S3/CloudFront, CORS and Cognito app client callback settings would need an explicit follow-up — that path is **out of scope** for the current internship stack.
{{% /notice %}}

---

Next: [5.7.4 - End-to-End Testing](../5.7.4-Test-end-to-end/)
