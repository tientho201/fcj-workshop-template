---
title: "Proposal"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# RAG Knowledge Assistant
## An AWS Serverless Solution for Internal Document Q&A

### 1. Executive Summary
The RAG Knowledge Assistant is an internal document Q&A chatbot built on **Retrieval-Augmented Generation (RAG)** architecture, designed to let employees upload documents (PDF, scanned images, plain text) and get answers grounded directly in that content instead of relying on an LLM's general pre-trained knowledge. The system runs entirely on AWS Serverless services — Lambda, SQS, Amazon Bedrock, and DynamoDB (storing both vectors and BM25 indices, replacing a dedicated search engine) — combined with Infrastructure as Code (Terraform) for reproducible, reviewable deployments. Beyond the core Q&A experience, the platform includes semantic caching to control Bedrock invocation costs, content moderation via Bedrock Guardrails, real-time operational monitoring, and an automated daily quality evaluation loop using the RAGAS framework.

### 2. Problem Statement
### What's the Problem?
Enterprise knowledge is often scattered across PDF and scanned documents, making manual lookup slow and repetitive. Off-the-shelf LLMs can answer fluently but are not grounded in an organization's actual internal content, risking confidently wrong (hallucinated) answers. There is also typically no systematic, quantitative way to measure whether a Q&A system's answers are actually trustworthy — teams tend to rely on subjective impressions rather than metrics.

### The Solution
The platform ingests documents through Amazon S3, buffers processing through Amazon SQS (with Dead Letter Queues for retry safety), and uses AWS Lambda together with Amazon Textract to digitize scanned files. Extracted content is chunked and embedded via Amazon Bedrock, then stored directly in Amazon DynamoDB as packed vectors and BM25 term-frequency data — a custom Python hybrid search layer (cosine similarity + BM25, fused via Reciprocal Rank Fusion) runs inside the Lambda itself, avoiding the cost and operational overhead of a dedicated search engine. User-facing queries flow through Amazon API Gateway (secured with Amazon Cognito), check an ElastiCache Serverless cache layer first to avoid redundant Bedrock calls, retrieve relevant context via this hybrid search, and generate answers via Amazon Bedrock (Claude 3 + Titan/Cohere Embeddings) filtered through Bedrock Guardrails for content moderation. Amazon CloudWatch, SNS, and AWS Chatbot classify and route operational alerts to Slack by severity, while an EventBridge Scheduler triggers a daily Lambda that runs RAGAS evaluation metrics (Faithfulness, Answer Relevancy, Context Precision) against recent conversations.

### Benefits and Return on Investment
The solution eliminates time-consuming manual document search by centralizing retrieval into one conversational interface, while semantic caching meaningfully cuts down repeated Bedrock invocation costs for semantically similar questions. Storing vectors and BM25 statistics directly in DynamoDB instead of provisioning a dedicated search engine (e.g., OpenSearch Serverless) removes an always-on baseline cost, keeping the retrieval layer effectively pay-per-use like the rest of the stack. Automated RAGAS evaluation replaces subjective "it seems fine" judgments with quantitative, trackable quality metrics, catching answer-quality regressions (e.g., low faithfulness) that are difficult to spot by manual review alone. Because the entire stack is serverless and provisioned through Terraform, the infrastructure can be torn down and rebuilt on demand (via dedicated teardown/rebuild scripts) when not actively in use, keeping operating costs low only while the stack is running rather than a fixed monthly baseline.

### 3. Solution Architecture
The platform is built around four independent but tightly coupled processing flows, all provisioned through Terraform for consistent, reviewable infrastructure changes:

![RAG Knowledge Assistant System Architecture Overview](/images/5-Workshop/5.1-Workshop-overview/aws-new.drawio.png)

### AWS Services Used
- **AWS Lambda**: Runs the document processing, chat engine, and RAGAS evaluation logic (Python 3.12).
- **Amazon S3**: Stores raw uploaded documents and RAGAS evaluation results.
- **Amazon SQS**: Buffers document processing events with a Dead Letter Queue for retry handling.
- **Amazon Textract**: Performs OCR on scanned files and images.
- **Amazon Bedrock**: Generates embeddings (Titan/Cohere) and answers (Claude 3), enforced through Guardrails for content moderation.
- **Amazon DynamoDB**: Stores document chunks (parent/child), packed vectors and BM25 term data for retrieval, chat history, and user feedback — replacing a dedicated search engine for semantic indexing.
- **Amazon API Gateway**: Exposes the chat, upload, and status endpoints.
- **Amazon Cognito**: Authenticates end users before granting API access.
- **Amazon ElastiCache Serverless**: Caches recent Q&A pairs to reduce latency and Bedrock cost.
- **Amazon CloudWatch**: Collects logs/metrics, custom dashboards, and alarms.
- **Amazon SNS + AWS Chatbot**: Routes severity-classified alerts to Slack.
- **Amazon EventBridge Scheduler**: Triggers the daily RAGAS evaluation job.
- **Terraform (HCP Terraform)**: Manages all infrastructure as code with remote state.

### Component Design
- **Flow 1 — Data Ingestion**: S3 receives uploads, an S3 Event triggers SQS, and a Lambda (Document Processor) extracts text (with Textract for scanned files), splits it into parent/child chunks, and generates embeddings + BM25 term data stored directly in DynamoDB — no external search engine involved.
- **Flow 2 — Realtime Q&A**: API Gateway (behind Cognito auth) invokes a Chat Engine Lambda that checks the cache, runs a custom Python hybrid search (cosine similarity + BM25 fused via Reciprocal Rank Fusion) against DynamoDB to retrieve context, calls Bedrock for answer generation through Guardrails, and writes the conversation to DynamoDB.
- **Flow 3 — Monitoring & Alert**: CloudWatch Alarms watch Lambda errors, API 5xx rates, DLQ depth, and Bedrock throttling, publishing to severity-based SNS topics routed to Slack via AWS Chatbot.
- **Flow 4 — RAG Evaluation**: An EventBridge-scheduled Lambda samples recent Q&A pairs daily, scores them with RAGAS metrics, and stores results in S3 while publishing summary scores to CloudWatch.

### 4. Technical Implementation
**Implementation Phases**
The project follows a 5-week build cycle after topic finalization, each phase building directly on the previous one:
- **Research & Architecture Design**: Finalize the RAG Knowledge Assistant topic, evaluate Serverless vs. managed alternatives (e.g., Bedrock Knowledge Bases), and produce a proposal with high-level and data-flow architecture diagrams.
- **Environment Setup**: Prepare the Terraform/IaC project structure, request Amazon Bedrock model access (Claude 3, Titan Embeddings), and configure the development environment.
- **Core Flow Development**: Implement Flow 1 (Data Ingestion) followed by Flow 2 (Realtime Q&A with Semantic Cache), validating each with hands-on testing before moving to the next.
- **Observability & Quality**: Implement Flow 3 (Monitoring & Alerting) and Flow 4 (RAGAS Evaluation) so the system can detect its own issues and measure its own answer quality.
- **Hardening & Delivery**: Tune retrieval parameters based on RAGAS findings, run load tests, audit IAM permissions, refactor Terraform into modules, finalize documentation, and deliver the live demo.

**Technical Requirements**
- **Account & Region**: An AWS account in **us-east-1 (N. Virginia)**, the region with full support for the required Bedrock models, plus an HCP Terraform account for remote state management.
- **Tooling**: Terraform 1.5+, AWS CLI v2, Python 3.12 (Lambda runtime), Git, and a code editor.
- **Permissions**: A deployment IAM policy scoped to exactly the service groups used (Lambda, S3, SQS/SNS/EventBridge, DynamoDB/ElastiCache, Bedrock/Textract, API Gateway/Cognito, CloudWatch/Chatbot, IAM role management) — least privilege throughout.
- **CI/CD**: GitHub Actions for checks/plan on every PR, with `terraform apply` gated behind a manual, reviewed trigger rather than running automatically on merge.

### 5. Timeline & Milestones
**Project Timeline** (5 weeks, following 2 weeks of AWS foundations training)
- **Week 1**: Topic ideation, finalize the RAG Knowledge Assistant proposal, design architecture diagrams, prepare the development environment.
- **Week 2**: Deliver Flow 1 — Data Ingestion end-to-end (S3 → SQS → Lambda → OCR → embeddings).
- **Week 3**: Deliver Flow 2 — Realtime Q&A with authenticated API, semantic cache, and Guardrails.
- **Week 4**: Deliver Flow 3 — Monitoring & Alerting, and Flow 4 — automated RAGAS evaluation.
- **Week 5**: Tune retrieval quality, load test, harden IAM, refactor IaC, finalize documentation, and present the live demo.

### 6. Budget Estimation
By storing vectors and BM25 data in DynamoDB (pay-per-request) instead of provisioning a dedicated search engine, the main always-on baseline cost driver is Amazon ElastiCache Serverless's minimum provisioned capacity, with Bedrock invocations metered on top.

### Infrastructure Costs
- **Estimated running cost**: ~$2.5/day while the stack is deployed (ElastiCache Serverless baseline capacity is the primary driver, with Lambda, S3, SQS, DynamoDB, and API Gateway usage-based costs adding comparatively little at this scale).
- **Cost controls in place**:
  - Using DynamoDB instead of a dedicated search engine for vector/BM25 storage avoids an additional always-on baseline cost for retrieval.
  - Exact-match caching reduces repeated Bedrock invocation costs for identical questions.
  - Dedicated teardown/rebuild scripts destroy the full stack (with a documents backup) when not actively in use, avoiding a fixed monthly baseline.
  - An AWS Budget alert monitors monthly spend independently of the main stack's lifecycle.

### 7. Risk Assessment
#### Risk Matrix
- Delayed Amazon Bedrock model access approval: Medium impact, medium probability.
- Low retrieval/answer quality (hallucination, poor context match): High impact, medium probability.
- Cost overrun from the always-on ElastiCache Serverless baseline: Medium impact, low probability.
- Misconfigured IAM permissions between services: High impact, low probability.
- Lambda concurrency limits under higher load: Medium impact, low probability.

#### Mitigation Strategies
- Bedrock access: Submit model access requests as early as possible in the setup phase, before development work depends on them.
- Retrieval quality: Establish the RAGAS evaluation loop early so quality regressions are caught with metrics, not guesswork, and can drive concrete tuning (chunk size, hybrid search weighting).
- Cost: AWS Budget alerts plus scripted teardown when the stack is idle.
- IAM: Apply least-privilege policies from the start and run a dedicated permissions audit before final delivery.
- Concurrency: Load-test before final demo to surface limits early and document scaling options.

#### Contingency Plans
- If Bedrock access is delayed, proceed with infrastructure and pipeline development using mocked embedding/generation calls, then integrate real model calls once access is granted.
- If costs approach budget thresholds, immediately run the teardown script to destroy non-critical resources.
- If retrieval quality cannot be sufficiently improved within the timeline, document the gap explicitly and scope it as a follow-up item rather than silently shipping degraded quality.

### 8. Expected Outcomes
#### Technical Improvements
Automated document ingestion and OCR replace manual document handling. Cached responses return in well under a second for repeated questions versus several seconds for a fresh Bedrock call. Automated RAGAS evaluation replaces subjective quality checks with daily, quantitative scoring. Real-time, severity-classified alerting shortens the time to detect operational issues.

#### Long-term Value
A reusable, documented reference architecture for Serverless GenAI systems on AWS, hands-on experience with Infrastructure as Code (Terraform) and event-driven design, and a foundation that can be extended toward broader enterprise knowledge management use cases.
