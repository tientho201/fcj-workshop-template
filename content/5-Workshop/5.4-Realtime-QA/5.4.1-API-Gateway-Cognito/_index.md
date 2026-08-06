---
title: "API Gateway and Cognito"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.4.1 </b> "
---

Stream 2's infrastructure is declared in `modules/query/main.tf`. This page presents the API Gateway and Cognito components — the communication gateway between users and the system.

#### 4 Routes Sharing 1 Lambda

Unlike defining each route individually, the project uses `for_each` on `local.api_routes` to create all 4 routes simultaneously, all pointing to the same `chat_engine` Lambda:

```hcl
locals {
  api_routes = {
    chat               = { path = "chat", method = "POST" }
    documents          = { path = "documents", method = "POST" }
    status             = { path = "status", method = "GET" }
    documents_decision = { path = "documents-decision", method = "POST" }
  }
}

resource "aws_api_gateway_resource" "routes" {
  for_each = local.api_routes
  rest_api_id = aws_api_gateway_rest_api.this.id
  parent_id   = aws_api_gateway_rest_api.this.root_resource_id
  path_part   = each.value.path_part
}

resource "aws_api_gateway_method" "routes" {
  for_each = local.api_routes

  rest_api_id      = aws_api_gateway_rest_api.this.id
  resource_id      = aws_api_gateway_resource.routes[each.key].id
  http_method      = each.value.method
  authorization    = "COGNITO_USER_POOLS"
  authorizer_id    = aws_api_gateway_authorizer.cognito.id
  api_key_required = var.api_require_api_key
}

resource "aws_api_gateway_integration" "routes" {
  for_each = local.api_routes
  rest_api_id = aws_api_gateway_rest_api.this.id
  resource_id = aws_api_gateway_resource.routes[each.key].id
  http_method = aws_api_gateway_method.routes[each.key].http_method
  # Lambda proxy integrations are always invoked with POST regardless of the
  # client's verb — this is the integration's own transport, not the route.
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.chat_engine.invoke_arn
}

resource "aws_lambda_permission" "apigw_invoke" {
  for_each = local.api_routes

  statement_id  = "AllowAPIGatewayInvoke${title(each.key)}"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.chat_engine.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.this.execution_arn}/*/${each.value.method}/${each.value.path_part}"
}
```

| Route                 | Method | Role                                                                                       |
| --------------------- | ------ | ------------------------------------------------------------------------------------------ |
| `/chat`               | POST   | Main RAG pipeline — receives questions, returns answers                                    |
| `/documents`          | POST   | Receives files from UI, writes to S3 `raw_documents` — hooks back into Stream 1's pipeline |
| `/status`             | GET    | UI polls ingestion progress (reads Stream 1's `ingestion_status` table)                    |
| `/documents-decision` | POST   | User confirms/cancels OCR — human-in-the-loop component connecting to Stream 1             |

#### Cognito Authentication and CORS Preflight

All 4 routes use `authorization = "COGNITO_USER_POOLS"` via a shared `aws_api_gateway_authorizer`. Only the `OPTIONS` method (CORS preflight) uses `authorization = "NONE"` with a `MOCK` integration:

```hcl
resource "aws_api_gateway_authorizer" "cognito" {
  name            = "${local.name_prefix}-cognito-authorizer"
  rest_api_id     = aws_api_gateway_rest_api.this.id
  type            = "COGNITO_USER_POOLS"
  provider_arns   = [aws_cognito_user_pool.users.arn]
  identity_source = "method.request.header.Authorization"
}

resource "aws_api_gateway_method" "options" {
  for_each = local.api_routes
  rest_api_id      = aws_api_gateway_rest_api.this.id
  resource_id      = aws_api_gateway_resource.routes[each.key].id
  http_method      = "OPTIONS"
  authorization    = "NONE"
  api_key_required = false
}


resource "aws_api_gateway_integration" "options" {
  for_each = local.api_routes
  rest_api_id = aws_api_gateway_rest_api.this.id
  resource_id = aws_api_gateway_resource.routes[each.key].id
  http_method = aws_api_gateway_method.options[each.key].http_method
  type        = "MOCK"
  request_templates = {
    "application/json" = jsonencode({ statusCode = 200 })
  }
}
```

{{% notice tip %}}
The `OPTIONS` method must use `authorization = "NONE"` because browsers send CORS preflight requests **without an `Authorization` header**. This is safe because the `MOCK` integration only returns empty headers, without invoking Lambda or accessing real data — it is not a security vulnerability.
{{% /notice %}}

#### Amazon Cognito User Pool

```hcl
resource "aws_cognito_user_pool" "users" {
  name = "${local.name_prefix}-users"

  password_policy {
    minimum_length    = 12
    require_lowercase = true
    require_uppercase = true
    require_numbers   = true
    require_symbols   = true
  }

  mfa_configuration = var.cognito_mfa_enabled ? "ON" : "OFF"

  dynamic "software_token_mfa_configuration" {
    for_each = var.cognito_mfa_enabled ? [1] : []
    content {
      enabled = true
    }
  }

  account_recovery_setting {
    recovery_mechanism {
      name     = "verified_email"
      priority = 1
    }
  }

  auto_verified_attributes = ["email"]

  tags = merge(var.tags, {
    Name = "${local.name_prefix}-users"
  })
}

resource "aws_cognito_user_pool_client" "app" {
  name         = "${local.name_prefix}-app-client"
  user_pool_id = aws_cognito_user_pool.users.id

  # Public client (SPA/mobile) — no client secret.
  generate_secret = false

  explicit_auth_flows = [
    "ALLOW_USER_PASSWORD_AUTH",
    "ALLOW_USER_SRP_AUTH",
    "ALLOW_REFRESH_TOKEN_AUTH",
  ]

  access_token_validity  = 60 # minutes
  id_token_validity      = 60 # minutes
  refresh_token_validity = 30 # days

  token_validity_units {
    access_token  = "minutes"
    id_token      = "minutes"
    refresh_token = "days"
  }

  prevent_user_existence_errors = "ENABLED"
}
```

Notable configuration points:

- **Public client, no client secret** — suitable for SPAs (Single Page Applications), as the frontend has no secure place to store secrets.
- **12-character password policy with all character types** (uppercase/lowercase/numbers/special characters).
- **Access/ID tokens valid for 60 minutes, refresh token for 30 days** — balances security and user experience (users do not have to log in too frequently).
- **MFA toggled via variable** `var.enable_mfa` — offers flexibility between dev environments (disabled for quick testing) and production (recommended enabled).

---

### Next Content

- [5.4.2 - Infrastructure: Semantic Cache, Guardrails and IAM](../5.4.2-Cache-Guardrails-IAM/)
