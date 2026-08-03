---
title: "Week 3 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives:

- Hold team meetings to discuss, propose, and finalize the topic for the individual project.

- Develop ideas and ensure the topic solves a practical problem on AWS.

- Research and select a suitable infrastructure architecture (preferably Serverless) to build the system.

- Complete the proposal with a clear technological direction and development roadmap.

- Visualize the solution using detailed architecture diagrams.

- Prepare the source code environment and troubleshoot issues related to the project template (Git submodule errors).

### Tasks to be carried out this week:

| Day | Task                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Start Date | Completion Date | Reference Material                                                                                                                                                                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2   | **Team meeting and project ideation:**<br>- Group meeting to discuss, propose, and determine the project topic.<br>- Analyzed internal knowledge retrieval and Q&A challenges (scattered information in PDF/scan files, slow search, time-consuming manual lookups).<br>- Referenced Amazon Bedrock Knowledge Bases and RAG (Retrieval-Augmented Generation) architecture to orient the idea.<br>- Agreed on and finalized the group project topic: RAG Knowledge Assistant.                                                                                                         | 07/06/2026 | 07/06/2026      | [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)<br> [What is RAG?](https://aws.amazon.com/what-is/retrieval-augmented-generation/) |
| 3   | **System research and design:**<br>- Discussed daily work plan with the team before starting.<br>- Explored benefits of Serverless architecture for GenAI systems (auto-scaling, cost optimization during idle time, no server management).<br>- Designed 4 core flows: Data Ingestion, Realtime Q&A (Semantic Cache), Monitoring/Alerting, and RAG Evaluation (RAGAS).<br>- Finalized technology stack: Serverless (Lambda, SQS, Bedrock, OpenSearch Serverless, DynamoDB) combined with IaC (Terraform).<br>- Consolidated and shared results with the team at the end of the day. | 07/07/2026 | 07/07/2026      | - [AWS Serverless Architecture](https://aws.amazon.com/serverless/)<br>- AWS Well-Architected – GenAI Lens                                                                                    |
| 4   | **Project Proposal writing:**<br>- Discussed daily work plan with the team before starting.<br>- Wrote project introduction, scope, and objectives (internal document Q&A chatbot with content moderation and answer quality measurement).<br>- Justified topic selection based on practical enterprise needs and GenAI/Serverless learning objectives.<br>- Defined specific use cases: document upload, real-time Q&A, user feedback collection, and automated RAG quality evaluation.<br>- Consolidated and shared progress with the team at the end of the day.                  | 07/08/2026 | 07/08/2026      | Consolidated knowledge from research documents on RAG and Bedrock system design.                                                                                                              |
| 5   | **Diagram creation:**<br>- Discussed daily work plan with the team before starting.<br>- Drew High-Level Architecture diagram covering 4 main pipelines: Ingestion, Realtime QA, Monitoring, and Evaluation.<br>- Drew Data Flow diagram describing how Serverless components (S3, SQS, Lambda, Bedrock, OpenSearch, DynamoDB, ElastiCache) interact to process documents and answer questions.<br>- Consolidated and shared results with the team at the end of the day.                                                                                                            | 07/09/2026 | 07/09/2026      |                                                                                                                                                                                               |
| 6   | **Environment initialization:**<br>- Discussed daily work plan with the team before starting.<br>- Inspected and reviewed initial project repository structure (Git repo, Terraform/IaC structure).<br>- Submitted model access requests on Amazon Bedrock (for Claude 3, Titan Embeddings).<br>- Resolved initial configuration issues (Git submodules, group IAM permissions, GitHub integration).<br>- Reconfigured accurately to prepare for the development phase.<br>- Consolidated and shared progress with the team at the end of the day.                                   | 07/10/2026 | 07/10/2026      |                                                                                                                                                                                               |

### Week 3 Achievements:

- **Team meeting & project finalization:** The work week started with a team meeting to discuss, propose, and orient the project topic. Following the discussion, the team finalized the ideation and selected the "RAG Knowledge Assistant" project—an internal document Q&A chatbot system built on RAG architecture. The idea originated from a practical problem: corporate information is often scattered across PDF/scanned documents, leading to time-consuming manual lookups and difficulty in leveraging existing knowledge.

- **Rationale for project selection:** During research, the team referenced Amazon Bedrock Knowledge Bases—AWS's managed RAG service. This confirmed that RAG addresses a genuine real-world demand recognized by AWS in the GenAI space. On this basis, the team decided that instead of using the ready-made managed service, the project would build the entire ingestion, retrieval, answer generation, and evaluation pipeline from scratch using Serverless architecture and Terraform (IaC). This approach aims to achieve three goals: master GenAI Serverless architecture on AWS, practice Infrastructure as Code skills, and maintain full control to customize retrieval, caching, and moderation logic according to custom needs.

- **Completion of Proposal documentation:** The team finalized a comprehensive Project Proposal detailing objectives, scope, and technical direction. Concurrently, high-level architecture and data flow diagrams were completed for all 4 primary pipelines (Ingestion, Realtime QA, Monitoring, RAGAS Evaluation), visualizing how Serverless services interact.

- **Development environment setup & troubleshooting:** Successfully resolved initial configuration issues in the repo/template, submitted Bedrock model access requests, and configured a synchronized development environment for the team, establishing readiness for infrastructure coding in subsequent weeks.

- **Effective team collaboration:** Maintained consistent daily collaboration habits throughout the week. Members discussed daily work plans each morning and reviewed accomplished results at the end of each day to keep everyone aligned on progress.
