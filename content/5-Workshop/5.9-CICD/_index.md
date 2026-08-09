---
title: "Building CI/CD"
date: 2026-08-09
weight: 9
chapter: false
pre: " <b> 5.9. </b> "
---

#### Context — Why Designed This Way

The infrastructure runs on **HCP Terraform** (workspace `RAGonAWS/RAG-app`, execution mode **"Remote"**) — meaning `terraform plan`/`apply` actually runs on HashiCorp workers, **not** on local machines or GitHub Actions runners. Therefore, CI/CD here **does not automatically run `terraform apply`** like a conventional build-deploy pipeline, but only **triggers** Terraform CLI commands, authenticating with a token (`TF_API_TOKEN`), and letting HCP Terraform handle the heavy lifting.

{{% notice warning %}}
📌 **The most critical cost constraint driving the entire design:** infrastructure is **proactively shut down** using `scripts/down.sh` when not in use to save costs (~$2.5/day when running) — so **merging code must not automatically apply**, preventing accidentally spinning up the entire shut-down stack.
{{% /notice %}}

2 workflows in `.github/workflows/`, running on GitHub Actions:

- **`ci.yml`** — runs on every PR and every push to `main`, consisting of **5 jobs** (4 run in parallel; `terraform-plan` waits on `terraform-checks` and only on PRs). Nothing is changed on AWS.
- **`deploy.yml`** — triggered **manually** (`workflow_dispatch`), which is where `terraform apply` actually runs.

#### Detailed Contents

1. [ci.yml — Checks on Every PR](5.9.1-CI-Workflow/)
2. [deploy.yml — Required Reviewer Issue and Fix](5.9.2-Deploy-Workflow/)
3. [Manual Setup and Scope Limitations](5.9.3-Manual-Setup-and-Scope-Limitations/)
