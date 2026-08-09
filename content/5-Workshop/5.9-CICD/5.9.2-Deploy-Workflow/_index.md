---
title: "deploy.yml — Required Reviewer Issue and Fix"
date: 2026-08-09
weight: 2
chapter: false
pre: " <b> 5.9.2. </b> "
---

This is the most interesting page in the CI/CD section — not because the design is complex, but because the **first design did not behave as intended**, and finding + fixing that is a real story worth documenting.

#### Original design (did not work)

The `terraform-apply` job was meant to use a GitHub Environment `production` with **"Required reviewers"** — idea: merge to `main` would **auto-trigger**, then **pause for approval** in the Actions tab before a real apply.

```yaml
# Original design (did NOT behave as expected)
on:
  push:
    branches: [main]

jobs:
  terraform-apply:
    environment: production # expected: wait for approval here
    runs-on: ubuntu-latest
    steps:
      - run: terraform apply -auto-approve
```

#### Broke in practice

{{% notice warning %}}
📌 **Deploy ran immediately — no approval pause.** Checking with `gh repo view --json isPrivate` showed why: **"Required reviewers"** on a GitHub Environment is **only available for private repos in an Organization on a paid GitHub Team/Enterprise plan**. `RAGonAWS` is a **private personal-account** repo, so that option **does not appear in the UI at all** — not a misconfiguration, a product-tier limit.
{{% /notice %}}

This is the hardest class of bug to spot: no error message, the YAML is valid, only the **behavior** is wrong — you only catch it by watching Deploy run instantly and then digging with `gh repo view`.

#### Fix: switch to `workflow_dispatch`

```yaml
# .github/workflows/deploy.yml (current)
on:
  workflow_dispatch: {} # see comments in deploy.yml:3-11

jobs:
  terraform-apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~> 1.9"
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
      - run: terraform init
      - run: terraform apply -auto-approve -input=false
```

Trigger changed from `push: branches: [main]` to `workflow_dispatch: {}` — **merging to `main` now does nothing for deploy**; you must open **Actions → Deploy → Run workflow** for a real apply.

The workflow also sets `concurrency: group: deploy-production` with `cancel-in-progress: false` so two applies never run at once against the same HCP Terraform workspace.

{{% notice note %}}
`workflow_dispatch` is the **free gate** that replaces Required reviewers, with the same intent: **no deliberate click → nothing gets applied**. You do not always need the “standard” feature — a simpler mechanism that meets the safety goal (no surprise auto-apply) is enough when the standard feature is unavailable.
{{% /notice %}}

![Deploy trigger before vs after the fix](../images/02-deploy-trigger-before-after.png)
*Before: push auto-ran apply (Required reviewer never appeared). After: manual “Run workflow” button in the Actions tab.*

---

Next: [5.9.3 - Manual Setup and Scope Limitations](../5.9.3-Manual-Setup-and-Scope-Limitations/)
