---
title: "Frontend"
date: 2026-08-07
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

#### RAG control panel

Unlike a typical SPA with a build step, this project's UI is a **single self-contained HTML/JS file** (`ui/index.html`) — **no React, no bundler, no `package.json`**. That fits the scope: an internal console to demo and exercise the RAG stack, not a public product that needs SEO or code-splitting.

The browser talks to the backend only through the **4 API routes from Stream 2** (see [5.4.1](../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)):

| Route | Purpose |
|---|---|
| `POST /chat` | Send questions, receive answers (+ `trace`, `cached`, `sources`) |
| `POST /documents` | Upload text (`content`) or binary (`content_base64`) |
| `GET /status` | Poll ingestion progress (`document_id`) |
| `POST /documents-decision` | Confirm OCR (`ocr`) or cancel for scanned PDFs |

The layout is **two panes**: **left** for actions (sign-in, upload, Q&A), **right** for observing the pipeline — step animation driven by **real timings** from the backend (`trace` / ingestion status), not staged effects.

{{% notice note %}}
📌 **Demo console, local only.** The stack does **not** host this file on Amplify or S3 static website hosting — open `ui/index.html` after Terraform apply (see [5.7.3](5.7.3-Deployment-Hosting/)).

📌 The UI labels the Redis step “Semantic cache”, but the implementation is an **exact-match** cache (normalized question hash) — same caveat as Stream 2. ElastiCache Serverless has no RediSearch/vector module here.

📌 Terraform defaults `api_require_api_key = true`, but `ui/index.html` does **not** send `x-api-key`. For the local UI to work, either set `api_require_api_key = false` in `terraform.tfvars`, or extend the UI to send `terraform output -raw api_key_value`.
{{% /notice %}}

#### UI overview

![Two-pane UI: action panel and pipeline observer](/images/5-Workshop/5.7-Web/image.png)
*Screenshot of the real two-pane console (left: connect / upload / chat; right: pipeline + log).*

#### Detailed contents

1. [Frontend Architecture and Authentication](5.7.1-Frontend-Architecture-Authentication/)
2. [Chat Interface and Document Upload](5.7.2-Chat-Upload-UI/)
3. [Deployment and Hosting](5.7.3-Deployment-Hosting/)
4. [End-to-End Testing](5.7.4-Test-end-to-end/)
