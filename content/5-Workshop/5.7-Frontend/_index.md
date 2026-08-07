---
title: "Frontend"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

#### Introduction

The Frontend is the web interface for end users: sign in, upload documents, ask questions, and view answers. It talks to the backend only through the **4 API routes from Stream 2** (see [5.4.1](../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)):

| Route | Purpose |
|---|---|
| `POST /chat` | Send questions, receive answers (+ `trace`, `cached`, `sources`) |
| `POST /documents` | Upload text (`content`) or binary (`content_base64`) |
| `GET /status` | Poll ingestion progress (`document_id`) |
| `POST /documents-decision` | Confirm OCR (`ocr`) or cancel for scanned PDFs |

The UI is a single file in the application repo (`ui/index.html`) and consists of 2 main parts:

- **Shell (HTML/CSS/JS)** — two-pane control panel. No React, no bundler, no `package.json`.
  - **Left pane:** (1) Connection — API URL, Cognito App Client ID, region, email/password; (2) Upload — file picker or paste text, OCR Yes/No box when needed; (3) Chat — messages, question box, **Phiên mới** (`session_id`).
  - **Right pane:** tabs for ingest vs query pipeline, step animation from real backend timings, plus a live log.
- **Integration with Cognito and API Gateway** — browser calls Cognito `InitiateAuth` (`USER_PASSWORD_AUTH`) for an ID token, then calls the four routes with `Authorization: <IdToken>`. Right-pane timings come from `/chat` `trace` or `/status` ingestion progress — not simulated.
  {{% notice note %}}
  📌 **Demo console, local only.** The stack does **not** host this file on Amplify or S3 static website hosting — open `ui/index.html` after Terraform apply (see [5.7.3](5.7.3-Deployment-Hosting/)).

  📌 The UI labels the Redis step “Semantic cache”, but the implementation is an **exact-match** cache (normalized question hash) — same caveat as Stream 2. ElastiCache Serverless has no RediSearch/vector module here.

  📌 Terraform defaults `api_require_api_key = true`, but `ui/index.html` does **not** send `x-api-key`. For the local UI to work, either set `api_require_api_key = false` in `terraform.tfvars`, or extend the UI to send `terraform output -raw api_key_value`.
  {{% /notice %}}

#### User Interface

![UI Interface](/images/5-Workshop/5.7-Web/image.png)

#### Detailed Contents

1. [Frontend Architecture and Authentication](5.7.1-Frontend-Architecture-Authentication/)
2. [Chat Interface and Document Upload](5.7.2-Chat-Upload-UI/)
3. [Deployment and Hosting](5.7.3-Deployment-Hosting/)
4. [End-to-End Testing](5.7.4-Test-end-to-end/)
