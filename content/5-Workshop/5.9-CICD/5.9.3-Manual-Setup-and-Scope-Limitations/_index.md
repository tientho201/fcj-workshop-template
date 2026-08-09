---
title: "Manual Setup and Scope Limitations"
date: 2026-08-09
weight: 3
chapter: false
pre: " <b> 5.9.3. </b> "
---

#### One-time setup (GitHub UI — cannot automate via code)

CI/CD only handles the **trigger**. Initial setup still needs manual steps in the GitHub UI:

1. **Add secret `TF_API_TOKEN`** — `Settings → Secrets and variables → Actions → New repository secret`. Create the token in HCP Terraform: `User Settings → Tokens` (or a **Team Token** for org `RAGonAWS` for tighter scope). The same secret feeds **both** workflows (`ci.yml` for `terraform plan`, `deploy.yml` for `terraform apply`).

![Adding TF_API_TOKEN in GitHub repository settings](../images/03-github-secret-tf-api-token.png)
*Settings → Secrets and variables → Actions, with `TF_API_TOKEN` added.*

{{% notice note %}}
📌 **No AWS access keys are needed in GitHub.** `terraform apply` runs on an **HCP Terraform worker** (Remote execution — see the [section overview](../)), not on the GitHub Actions runner. AWS credentials live in **HCP Terraform workspace variables**, configured earlier and fully separate from GitHub Actions. That is a useful side effect of Remote execution: the attack surface on the GitHub side is smaller than putting AWS keys directly into GitHub Secrets.
{{% /notice %}}

#### Out of scope for the current CI/CD

{{% notice warning %}}
These points are **intentionally not automated** — document them as scope limits, not forgotten work:

- **`scripts/up.sh` / `scripts/down.sh`** (recreate Cognito users, restore document backups when spinning infrastructure up/down for cost control — see the [section overview](../)) still run **manually**. CI/CD only covers `terraform apply` for infrastructure, not re-seeding data/accounts after a rebuild.
- **`handler.py` for both Lambdas** (real `boto3` / Bedrock / Cognito / S3 / SQS paths) is **not** in the unit suite. `python-test` covers **33** tests over pure modules + shared-copy drift (`chunking`, `bm25`, `vector_store`, `retrieval` with fake DynamoDB, `tracing`/`embeddings` sync) — see [5.9.1](../5.9.1-CI-Workflow/) and [5.10.3](../../5.10-System-Testing/5.10.3-Layer-3-Automated-Unit-Testing/).
{{% /notice %}}

#### End-to-end CI/CD flow

```
PR opened/updated → ci.yml
                    (secret-scan, python-lint, python-test, terraform-checks
                     in parallel; terraform-plan after checks, PR only)
                  → code review

Merge to main     → no auto-deploy (push trigger removed)

Real deploy       → Actions → "Deploy" → "Run workflow"
                  → deploy.yml → terraform apply -auto-approve
                  (runs on HCP Terraform worker; AWS creds from
                   workspace variables)
```

{{% notice tip %}}
A longer write-up also lives in `docs/CI-CD.md` in the app repo — useful when you add real screenshots/numbers. Prefer the workflows under `.github/workflows/` as source of truth if that doc lags (e.g. after `python-test` was added).
{{% /notice %}}

#### Checklist — CI/CD done

- [ ] Clear why `deploy.yml` does not run on merge (cost / `down.sh`, not a design bug)
- [ ] Clear why Required reviewers were unavailable on a personal private repo, and how `workflow_dispatch` replaces them
- [ ] `TF_API_TOKEN` secret added; `ci.yml` can run `terraform plan` on a trial PR
- [ ] `deploy.yml` run at least once via “Run workflow”; apply succeeded
- [ ] Scope limits recorded: `up.sh`/`down.sh` manual; know `python-test` = 33 pure-logic tests (not `handler.py`); gap pointed at 5.10.3
