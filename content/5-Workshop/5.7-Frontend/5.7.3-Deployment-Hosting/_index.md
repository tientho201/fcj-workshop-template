---
title: "Deployment and Hosting"
date: 2026-08-07
weight: 3
chapter: false
pre: " <b> 5.7.3. </b> "
---

#### No hosting infrastructure

{{% notice warning %}}
📌 Unlike a typical S3 + CloudFront + OAC SPA design, this project **does not provision any hosting for the UI** — nothing in the `.tf` files (no `cloudfront` / OAC resources for the console).
{{% /notice %}}

Because `ui/index.html` is a self-contained static file with no build step, the simplest — and current — “deploy” path is to open the file in a browser:

```bash
# from the RAGonAWS repo root
open ui/index.html          # macOS
# or double-click the file / start ui/index.html on Windows
```

No local Node server, no domain, no SPA routing (one page, nothing to 404).

#### Runtime config

With no build step, environment values (API endpoint, Cognito Client ID, region) are **not baked in at build time**. Fill section **1 · Kết nối** whenever those values change (API Gateway and Cognito App Client get new IDs after a full recreate). Values persist in `localStorage` for the next open:

```bash
terraform output api_gateway_endpoint_url
terraform output cognito_app_client_id
# only if api_require_api_key = true (UI does not send this by default):
# terraform output -raw api_key_value
```

{{% notice tip %}}
API Gateway / Cognito Client IDs change whenever you `terraform apply` from a full recreate (destroy + create, not an in-place update). **Always re-run the two `terraform output` commands above after a full stack rebuild**, then update the Connection panel — otherwise the UI calls a dead endpoint.
{{% /notice %}}

If API key is required, either set `api_require_api_key = false` in `terraform.tfvars` or extend the UI (see [5.7.1](../5.7.1-Frontend-Architecture-Authentication/)).

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

| Limit            | Detail                                                                                               |
| ---------------- | ---------------------------------------------------------------------------------------------------- |
| API Gateway body | 10 MB max; base64 PDF/images inflate ~33%, so usable binary size is smaller                          |
| Multi-page PDF   | Sync Textract (`detect_document_text`) handles **1 page** only — multi-page fails with a clear error |
| Bedrock quota    | Bursting requests can return HTTP 429 (`retryable`) — wait and retry                                 |
| Hosting          | Frontend is not part of the Terraform destroy/apply surface; only open the file locally              |
| Layout           | Below ~900px width the grid collapses to one column                                                  |

#### Possible follow-up (not built)

If the UI must be shared with people who cannot run `terraform output`, a reasonable path is **one S3 bucket** (static website or CloudFront + OAC origin) hosting this single HTML file — no build pipeline needed.

{{% notice note %}}
That path is **out of scope** for the current internship stack. In a report, treat it as a possible next step — not as completed work. Hosting later would also need explicit CORS and Cognito app-client settings.
{{% /notice %}}

---

#### Next content

- [5.7.4 - End-to-End Testing](../5.7.4-Test-end-to-end/)
