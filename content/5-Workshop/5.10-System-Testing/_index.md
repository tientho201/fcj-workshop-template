---
title: "System Testing"
date: 2024-01-01
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---

{{% notice note %}}
📌 **This is a summary page**, which does not repeat details already present in individual sections — each row in the table below links directly to where it is described in full. If an independent "Testing" section is required for a report, this is the exact page to reference: it provides a comprehensive overview without needing to re-read 8 separate pages.
{{% /notice %}}

#### Summary: What was tested in each section and how

| Section | What was tested | How it was tested | View Details |
| --- | --- | --- | --- |
| Stream 1 — Ingestion | 4 file types (text/PDF text/PDF scan/image), OCR confirmation, resume/cancel | Manual via UI | [5.3.6](../5.3-Data-Ingestion/5.3.6-End-To-End-Testing/) |
| Stream 2 — Realtime QA | 11 scenarios: cache, rewrite, retrieval, guardrail, throttle, feedback | Manual via UI + DevTools + script | [5.4.8](../5.4-Realtime-QA/5.4.8-End-To-End-Testing/) |
| Stream 3 — Monitoring | 8 scenarios: 4 two-way alarms (ALARM↔OK), subscription, Slack OAuth | Manual, simulated errors | [5.5.4](../5.5-Monitoring/5.5.4-End-to-End-Testing/) |
| Stream 4 — RAGAS | Data architecture (GSI, IAM) verified; **RAGAS logic not tested with real data, not deployed** | Not testable yet (not deployed) | [5.6.4](../5.6-RAGAS/5.6.4-Testing-Deployment-Notes/) |
| Frontend | 11 scenarios: login, upload, OCR dialog, responsive | Manual via browser | [5.7.4](../5.7-Frontend/5.7.4-Test-end-to-end/) |
| Backend (pure logic) | **33 unit tests** (chunking, BM25, vector store, RRF, 4 shared-file drift) | **Automated**, runs in CI | [5.8.4](../5.8-Backend/5.8.4-Backend-Testing/) |
| CI — static analysis | Linting (`ruff`), Terraform syntax (`fmt`/`validate`), secret scanning (`gitleaks`) | **Automated**, every PR/push | [5.9.1](../5.9-CICD/5.9.1-CI-Workflow/) |
| Manual/E2E Testing | 3 hand-crafted PDF types, real API invocation script, **1 real bug caught** (504 → missing VPC endpoint) | Manual, not automatically repeatable | [5.10.1](5.10.1-Manual-E2E-Testing/) |

#### At a Glance: Two Fundamentally Different Testing Types

Looking across the table above, all testing in the project falls into exactly 2 types — which must be clearly distinguished when presenting a report, rather than grouped under "fully tested":

| | Automated (Repeatable) | Manual (Non-repeatable) |
| --- | --- | --- |
| **Includes** | Lint/validate/secret-scan (CI) + 33 unit tests (pytest) | All remaining 11+8+11+4 scenarios in the table above, plus E2E story in [5.10.1](5.10.1-Manual-E2E-Testing/) |
| **Re-run on every push?** | ✅ | ❌ |
| **Verifies real business logic?** | Partial — pure Python logic only, no `boto3`/AWS calls | ✅ — runs on real AWS infrastructure |

{{% notice warning %}}
**Actual Remaining Gap** (not speculation, confirmed in [5.8.4](../5.8-Backend/5.8.4-Backend-Testing/)): `handler.py` in both Lambdas — the portion **directly calling `boto3`/AWS** — lacks automated testing at any layer, verified only through manual (non-repeatable) testing. The `/feedback` route and GSI join via `message_id-index` share the same status. This is a point to state directly under the "Limitations" section of any report.
{{% /notice %}}

#### Detailed Contents (Non-overlapping with above)

1. [Manual/E2E Testing](5.10.1-Manual-E2E-Testing/)
