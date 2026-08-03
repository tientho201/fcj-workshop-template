---
title: "Week 7 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7 Objectives:

- Review all 4 system flows from a holistic perspective, resolve coarse edges identified in previous weeks (especially retrieval quality), write operational documentation, and deliver the official project demo to the team/mentor.

### Tasks to be carried out this week:

| Day | Task                                                                                                                                                                                                                                                                                                                                                                                                                               | Start Date | Completion Date | Reference Material                                                                                    |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | ----------------------------------------------------------------------------------------------------- |
| 2   | **Retrieval Quality Improvement:**<br>- Held early-week meeting to review low Faithfulness issues identified last week.<br>- Adjusted Hybrid Search parameters in OpenSearch (increased Vector Search weight vs BM25 from 50/50 ratio to 70/30).<br>- Reduced child chunk size from 300 to 200 tokens to increase retrieval precision.<br>- Re-ran 5 previously low-scoring questions — 4 out of 5 showed significant improvement. | 08/03/2025 | 08/03/2025      | Personal Project                                                                                      |
| 3   | **Preliminary Load Testing & IAM Audit:**<br>- Used a script sending 50 concurrent requests to API Gateway to test system reaction.<br>- Observed ElastiCache Serverless handled load smoothly while Lambda Chat Engine hit default concurrency limit (1000) — noted for future scaling.<br>- Performed final audit of all IAM Policies following least privilege principles.                                                      | 08/04/2025 | 08/04/2025      | [AWS Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html) |
| 4   | **Terraform & IaC Refactoring:**<br>- Cleaned up Terraform codebase, modularized by system flows (ingestion, chat-api, monitoring, evaluation), wrote deployment README.<br>- Re-inspected manually created AWS resources from prior fast-paced iterations and imported them into Terraform state.                                                                                                                                 | 08/05/2025 | 08/05/2025      | [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)                             |
| 5   | **Operational Documentation & Runbook:**<br>- Authored overall architecture runbook, alert resolution guide (e.g., DLQ Depth > 0 troubleshooting and manual requeueing), and maintenance checklist.<br>- Prepared demo slides and script for final presentation.                                                                                                                                                                   | 08/06/2025 | 08/06/2025      | Personal Project                                                                                      |
| 6   | **Official Demo & Project Summary:**<br>- Presented full project to team/mentor: live end-to-end execution from document upload → Q&A → simulated alerts → RAGAS evaluation report.<br>- Collected feedback (expanding test query dataset, adding API rate-limiting).<br>- Summarized the 5-week journey and outlined future enhancement directions.                                                                               | 08/07/2025 | 08/07/2025      | Personal Project                                                                                      |

### Week 7 Achievements — and Project Summary:

- **Metrics-Driven System Fine-Tuning:** Tuning Hybrid Search weights (Vector 70 / BM25 30) and child chunking (300 → 200 tokens) validated the RAGAS evaluation loop established in Week 6, yielding noticeable score boosts on 4/5 problematic queries.
- **Load Testing & IAM Hardening:** Verified ElastiCache Serverless stability under 50 concurrent requests, identified Lambda concurrency limits for future scaling, and finalized least-privilege IAM policies.
- **Complete Infrastructure as Code (IaC):** Modularized Terraform scripts across all 4 system flows, successfully imported manual console resources into state, and provided complete setup instructions.
- **Operational Runbook & Successful Final Demo:** Delivered operational documentation and conducted a seamless live demo spanning document processing, chat response, alarm triggering, and automated RAGAS quality reporting.

---

### Overall Summary of the RAG Knowledge Assistant Project:

While Week 7 did not introduce many new technologies, it focused on stabilization, refactoring, and bringing the entire architecture into an operational, long-term state.

Reflecting on the 5-week implementation journey (from Week 3 kickoff to Week 7 completion), the RAG Knowledge Assistant covered the full lifecycle of an enterprise GenAI system: architecture design, data pipelines, AI model integration, API delivery, operational monitoring, and automated quality benchmarking. The primary takeaway lies in mastering serverless event-driven architecture and relying on quantitative metrics to drive AI system improvements.
