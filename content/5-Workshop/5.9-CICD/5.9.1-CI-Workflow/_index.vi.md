---
title: "ci.yml — Kiểm tra trên mọi PR"
date: 2026-08-09
weight: 1
chapter: false
pre: " <b> 5.9.1. </b> "
---

`ci.yml` chạy trên **mọi PR** vào `main` (và mọi push vào `main`). Có **5 job độc lập**. Bốn job khởi động song song; `terraform-plan` chờ `terraform-checks` và **chỉ chạy trên pull request**.

| Job                | Việc làm                                                                                                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `secret-scan`      | `gitleaks` quét **toàn bộ lịch sử commit** của PR (`fetch-depth: 0`) — bắt sớm nếu có key/credential lỡ tay commit                                                         |
| `python-lint`      | `ruff check modules` — chỉ soi code Lambda tự viết; thư viện `pypdf` đã vendor được loại trừ (`ruff.toml`)                                                                 |
| `python-test`      | `pytest` (xem bảng dưới) — **33** test logic thuần Python; không cần AWS credentials. Chi tiết: [5.10.3](../../5.10-System-Testing/5.10.3-Layer-3-Automated-Unit-Testing/) |
| `terraform-checks` | `terraform fmt -check` + `terraform validate`, dùng `terraform init -backend=false` — không cần đăng nhập HCP Terraform, chỉ tải provider để kiểm tra cú pháp              |
| `terraform-plan`   | `terraform plan` **thật**, remote trên HCP Terraform (cần `TF_API_TOKEN`) — **chỉ trên PR**, chỉ là **speculative plan**, không đổi gì trên AWS                            |

#### `python-test` chạy những gì

Thư mục `tests/` import source Lambda **theo đường dẫn** (`conftest.load_module` / `importlib`) vì `modules/*/lambda_src/*/` là layout zip phẳng — không có `__init__.py`, import sibling tên trần. Dependency dev chỉ có `pytest` trong `tests/requirements.txt` (không đóng vào zip Lambda).

| File test                           | Code đang kiểm                    | Kiểm tra gì                                                                                 |
| ----------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------- |
| `test_chunking.py` (8)              | `document_processor/chunking.py`  | Tách parent–child, ID, overlap, biên đoạn văn                                               |
| `test_bm25.py` (9)                  | `chat_engine/bm25.py`             | Tokenize (kể cả dấu tiếng Việt), IDF ≥ 0, scoring                                           |
| `test_vector_store.py` (6)          | `chat_engine/vector_store.py`     | L2 normalize, pack/unpack float32, cosine = dot product                                     |
| `test_retrieval.py` (6)             | `chat_engine/retrieval.py`        | RRF, hybrid search, fetch parent — bảng DynamoDB **fake**, không AWS thật                   |
| `test_shared_copies_in_sync.py` (4) | File dùng chung **cả hai** Lambda | Chống drift: logic `bm25` / `vector_store` + `tracing.py` / `embeddings.py` giống từng byte |

```yaml
# .github/workflows/ci.yml (rút gọn)
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
`terraform-plan` khai báo `needs: terraform-checks` — chỉ chạy plan thật (**tốn phút chạy trên HCP Terraform**, có giới hạn/tính phí) **sau khi** `fmt`/`validate` đã sạch, tránh lãng phí cho lỗi cú pháp lẽ ra bắt được sớm hơn. Workflow còn dùng `concurrency` với `cancel-in-progress: true` để run PR cũ bị thay thế không chồng chất.
{{% /notice %}}

---

#### Nội dung tiếp theo

- [5.9.2 - deploy.yml — Vấn đề Required reviewer và cách sửa](../5.9.2-Deploy-Workflow/)
