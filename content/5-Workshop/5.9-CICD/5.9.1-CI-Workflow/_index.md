---
title: "ci.yml — Checks on Every PR"
date: 2026-08-09
weight: 1
chapter: false
pre: " <b> 5.9.1. </b> "
---

`ci.yml` runs on **every PR** to `main` (and every push to `main`). It defines **5 independent jobs**. Four start in parallel; `terraform-plan` waits for `terraform-checks` and runs **only on pull requests**.

| Job                | What it does                                                                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `secret-scan`      | `gitleaks` scans the **full commit history** of the PR (`fetch-depth: 0`) — catch keys/credentials committed by mistake early                                  |
| `python-lint`      | `ruff check modules` — lint Lambda code only; vendored `pypdf` is excluded (`ruff.toml`)                                                                       |
| `python-test`      | `pytest` (see table below) — **33** pure-Python tests; no AWS credentials. Details: [5.10.3](../../5.10-System-Testing/5.10.3-Layer-3-Automated-Unit-Testing/) |
| `terraform-checks` | `terraform fmt -check` + `terraform validate`, using `terraform init -backend=false` — no HCP login; just download providers for syntax checks                 |
| `terraform-plan`   | Real `terraform plan` remote on HCP Terraform (needs `TF_API_TOKEN`) — **PR only**, **speculative** (changes nothing on AWS)                                   |

#### What `python-test` actually runs

`tests/` imports Lambda source **by path** (`conftest.load_module` / `importlib`) because `modules/*/lambda_src/*/` are flat zip layouts — no `__init__.py`, bare sibling imports. Dev dependency is only `pytest` in `tests/requirements.txt` (never vendored into a Lambda zip).

| Test file                           | Code under test                  | What it checks                                                                             |
| ----------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `test_chunking.py` (8)              | `document_processor/chunking.py` | Parent–child split, IDs, overlap, paragraph boundaries                                     |
| `test_bm25.py` (9)                  | `chat_engine/bm25.py`            | Tokenize (incl. Vietnamese diacritics), IDF ≥ 0, scoring                                   |
| `test_vector_store.py` (6)          | `chat_engine/vector_store.py`    | L2 normalize, float32 pack/unpack, cosine via dot product                                  |
| `test_retrieval.py` (6)             | `chat_engine/retrieval.py`       | RRF fusion, hybrid search, parent fetch — **fake** DynamoDB tables, no real AWS            |
| `test_shared_copies_in_sync.py` (4) | Shared files in **both** Lambdas | Drift guard: `bm25` / `vector_store` logic + byte-identical `tracing.py` / `embeddings.py` |

```yaml
# .github/workflows/ci.yml (abbreviated)
jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2

  python-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install ruff
      - run: ruff check modules

  python-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r tests/requirements.txt
      - run: pytest

  terraform-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "~> 1.9" }
      - run: terraform fmt -check -recursive -diff
      - run: terraform init -backend=false
      - run: terraform validate

  terraform-plan:
    needs: terraform-checks
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "~> 1.9"
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
      - run: terraform init
      - run: terraform plan -input=false -no-color
```

{{% notice tip %}}
`terraform-plan` declares `needs: terraform-checks` — the real plan (**uses HCP Terraform run minutes**, which have limits/cost) only runs **after** `fmt`/`validate` are clean, so syntax errors are caught cheaper and earlier. The workflow also uses `concurrency` with `cancel-in-progress: true` so superseded PR runs do not pile up.
{{% /notice %}}

---

#### Next Content

- [5.9.2 - deploy.yml — Required Reviewer Issue and Fix](../5.9.2-Deploy-Workflow/)
