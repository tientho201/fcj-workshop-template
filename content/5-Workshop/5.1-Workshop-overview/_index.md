---
title: "Introduction"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.1. </b> "
---

#### Design Principles

The **RAG Knowledge Assistant** system is designed based on four core principles:

- **Serverless-first** — zero server management, automatic scaling based on workload, pay-as-you-go pricing model (S3, Lambda, SQS, DynamoDB, OpenSearch Serverless, and ElastiCache Serverless are all fully serverless/managed services).
- **Event-driven** — system components communicate asynchronously via events (S3 Events, SQS messages) instead of direct synchronous calls, ensuring higher fault tolerance and independent scalability per component.
- **Separation of Concerns** — four main processing flows operate independently; each flow can be developed, tested, and deployed individually without affecting the others.
- **Observable by Design** — every component pushes logs/metrics directly to CloudWatch from day one, eliminating the need to add monitoring reactively after incidents occur.
  The entire infrastructure is managed using **Terraform (Infrastructure as Code)**, ensuring consistent environment reproducibility (dev/staging/prod) and enabling infrastructure change reviews via Pull Requests.

#### System Architecture Overview

<div align="center">

![RAG Knowledge Assistant System Architecture Overview](/images/5-Workshop/5.1-Workshop-overview/aws-new.drawio.png)

<p style="font-size: 0.85em; color: #666; font-style: italic; margin-top: 8px;">
 System Architecture Overview of RAG Knowledge Assistant
</p>

</div>

The architecture consists of four main processing flows operating independently while staying tightly connected through shared data stores (vector index, chat history). Detailed breakdowns of each flow will be presented in subsequent chapters — this section summarizes their roles and key components to provide a holistic view prior to implementation.

| Flow | Role | Key Components |
| --- | --- | --- |
| **1. Data Ingestion** | Ingest, digitize (OCR), and semantically index documents | S3, SQS (+ DLQ), Lambda, Textract, Bedrock (Embedding), OpenSearch Serverless |
| **2. Realtime Q&A** | Receive queries, retrieve context, generate answers, cache results | API Gateway, Cognito, Lambda, ElastiCache Serverless, OpenSearch Serverless, Bedrock (Claude/Titan + Guardrails), DynamoDB |
| **3. Monitoring & Alert** | System monitoring, severity-based alerting | CloudWatch (Logs, Metrics, Alarms, Dashboard), SNS, AWS Chatbot, Slack |
| **4. RAG Evaluation** | Daily automated evaluation of answer quality | EventBridge Scheduler, Lambda (RAGAS Runner), S3 (Evaluation Results) |

#### AWS Services Summary Table

| Category | Service |
| --- | --- |
| Compute | AWS Lambda |
| Storage | Amazon S3 |
| Messaging | Amazon SQS, Amazon SNS |
| Database | Amazon DynamoDB, Amazon ElastiCache Serverless |
| Search & AI | Amazon OpenSearch Serverless, Amazon Bedrock, Amazon Textract |
| API & Security | Amazon API Gateway, Amazon Cognito, IAM |
| Monitoring & Operations | Amazon CloudWatch, AWS Chatbot |
| Orchestration | Amazon EventBridge Scheduler |

In the next chapter, we will prepare the necessary AWS account and environment setup before diving into the step-by-step deployment of each flow.
