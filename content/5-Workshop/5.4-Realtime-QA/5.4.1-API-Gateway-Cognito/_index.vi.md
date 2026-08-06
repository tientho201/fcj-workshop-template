---
title: "API Gateway và Cognito"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.4.1 </b> "
---

Hạ tầng của Luồng 2 được khai báo tại `modules/query/main.tf`. Trang này trình bày phần API Gateway và Cognito — cổng giao tiếp giữa người dùng và hệ thống.

#### 4 route dùng chung 1 Lambda

Khác với cách khai báo từng route riêng lẻ, dự án dùng `for_each` trên `local.api_routes` để tạo cả 4 route cùng lúc, tất cả trỏ về chung 1 Lambda `chat_engine`:

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

| Route                 | Method | Vai trò                                                                          |
| --------------------- | ------ | -------------------------------------------------------------------------------- |
| `/chat`               | POST   | Pipeline RAG chính — nhận câu hỏi, trả câu trả lời                               |
| `/documents`          | POST   | Nhận file từ UI, ghi vào S3 `raw_documents` — khớp lại vào đúng pipeline Luồng 1 |
| `/status`             | GET    | UI poll tiến trình ingestion (đọc bảng `ingestion_status` của Luồng 1)           |
| `/documents-decision` | POST   | Người dùng xác nhận/hủy OCR — phần human-in-the-loop nối sang Luồng 1            |

#### Xác thực Cognito và CORS preflight

Cả 4 route đều dùng `authorization = "COGNITO_USER_POOLS"` qua 1 `aws_api_gateway_authorizer` chung. Riêng method `OPTIONS` (CORS preflight) dùng `authorization = "NONE"` kèm tích hợp `MOCK`:

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
Method `OPTIONS` phải dùng `authorization = "NONE"` vì trình duyệt gửi CORS preflight **không kèm `Authorization` header**. Điều này an toàn vì tích hợp `MOCK` chỉ trả về header rỗng, không chạm vào Lambda hay dữ liệu thật - không phải lỗ hổng bảo mật.
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

Điểm cấu hình đáng chú ý:

- **Public client, không có client secret** — phù hợp cho SPA (Single Page Application), vì frontend không có nơi lưu secret an toàn.
- **Password policy 12 ký tự, đủ loại** (hoa/thường/số/ký tự đặc biệt).
- **Access/ID token sống 60 phút, refresh token 30 ngày** — cân bằng giữa bảo mật và trải nghiệm người dùng (không phải đăng nhập lại quá thường xuyên).
- **MFA bật/tắt qua biến** `var.enable_mfa` — linh hoạt giữa môi trường dev (tắt để test nhanh) và production (nên bật).

---

#### Nội dung tiếp theo

- [5.4.2 - Hạ tầng: Semantic Cache, Guardrails và IAM](../5.4.2-Cache-Guardrails-IAM/)
