---
title: "Manual Setup and Scope Limitations"
date: 2026-08-09
weight: 3
chapter: false
pre: " <b> 5.9.3. </b> "
---

#### One-Time Setup (In GitHub UI, Not Automated via Code)

CI/CD only handles the **trigger** — initial setup still requires manual steps in the GitHub UI:

1. **Add secret `TF_API_TOKEN`** — `Settings → Secrets and variables → Actions → New repository secret`. Retrieve the token from HCP Terraform: `User Settings → Tokens` (or a **Team Token** for org `RAGonAWS` for tighter permissions). This secret is **shared by both workflows** (`ci.yml` for `terraform plan`, `deploy.yml` for `terraform apply`).

![Add secret 1](/images/5-Workshop/5.9-CICD/image5.9.3-1.png)
![Add secret 2](/images/5-Workshop/5.9-CICD/image5.9.3-2.png)

{{% notice note %}}
📌 **No AWS access keys need to be added to GitHub.** `terraform apply` runs on an **HCP Terraform worker** (Remote execution mode — see [overview page](../)), not on the GitHub Actions runner. AWS credentials reside in **workspace variables within HCP Terraform**, pre-configured and isolated from GitHub Actions. This is a side benefit of Remote execution: the attack surface on the GitHub side is smaller compared to placing AWS credentials directly as GitHub Secrets.
{{% /notice %}}

#### Out of Scope for Current CI/CD

{{% notice warning %}}
The following points are **intentionally not automated** — documented clearly as scope limitations, not forgotten work:

- **`scripts/up.sh` / `scripts/down.sh`** (recreating Cognito accounts, seeding document backups when toggling infrastructure for cost constraints — see [overview page](../)) **still run manually**. CI/CD only manages infrastructure `terraform apply`, not re-initializing data/accounts after rebuilding the stack.
- **`handler.py` of both Lambdas** (real `boto3` / Bedrock / Cognito / S3 / SQS paths) is **not yet** in the unit suite. `python-test` covers **33** tests over pure modules + duplicate copy drift prevention (`chunking`, `bm25`, `vector_store`, `retrieval` with fake DynamoDB, `tracing`/`embeddings` sync) — see [5.9.1](../5.9.1-CI-Workflow/).
  {{% /notice %}}

#### CI/CD Flow Summary

```
PR opened/updated → ci.yml
                    (secret-scan, python-lint, python-test, terraform-checks
                     in parallel; terraform-plan after checks, PR only)
                  → code review

Merge to main     → NO auto-deploy (push trigger removed)

Real deploy       → Actions → "Deploy" → "Run workflow"
                  → deploy.yml → terraform apply -auto-approve
                  (runs on HCP Terraform worker; AWS creds from
                   workspace variables)
```

{{% notice tip %}}
A longer description is also available in `docs/CI-CD.md` in the application repo — useful when attaching real images/metrics. If that document lags in time (e.g. before `python-test` was added), take `.github/workflows/` as the source of truth.
{{% /notice %}}
