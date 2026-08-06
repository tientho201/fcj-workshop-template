---
title: "Frontend"
date: 2024-01-01
weight: 7
chapter: false
pre: " <b> 5.7. </b> "
---

#### Introduction

The Frontend is the web interface for end users to interact with the system: logging in, uploading documents, asking questions, and viewing answers. The frontend communicates with the backend entirely through **4 API routes built in Stream 2** (see [page 5.4.1](../5.4-Realtime-QA/5.4.1-API-Gateway-Cognito/)):

| Route                      | Purpose                                  |
| -------------------------- | ---------------------------------------- |
| `POST /chat`               | Send questions, receive answers          |
| `POST /documents`          | Upload new documents                     |
| `GET /status`              | Poll document processing progress        |
| `POST /documents-decision` | Confirm/cancel OCR for scanned documents |

#### RAG Control Panel

The project's interface is **a self-contained pure HTML/JS file** (`ui/index.html`) — **no React, no bundler, no `package.json`**. This choice fits the project scale: an internal dashboard for demoing/testing the RAG system, not a public product requiring SEO or code-splitting.

#### User Interface

![UI Interface](/images/5-Workshop/5.7-Web/image.png)

#### Detailed Contents

1. [Frontend Architecture and Authentication](5.7.1-Frontend-Architecture-Authentication/)
2. [Chat Interface and Document Upload](5.7.2-Chat-Upload-UI/)
3. [Deployment and Hosting](5.7.3-Deployment-Hosting/)
4. [End-to-End Testing](5.7.4-Test-end-to-end/)
