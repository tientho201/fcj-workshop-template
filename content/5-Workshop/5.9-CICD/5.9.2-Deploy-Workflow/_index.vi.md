---
title: "deploy.yml — Vấn đề Required reviewer và cách sửa"
date: 2026-08-09
weight: 2
chapter: false
pre: " <b> 5.9.2. </b> "
---

Đây là trang thú vị nhất trong phần CI/CD — không phải vì thiết kế phức tạp, mà vì **thiết kế ban đầu không hoạt động như dự tính**, và quá trình phát hiện + sửa là câu chuyện thật đáng ghi lại.

#### Thiết kế ban đầu (không hoạt động)

Job `terraform-apply` dự kiến gắn **GitHub Environment `production`** với **"Required reviewer"** — ý tưởng: merge vào `main` sẽ **tự trigger**, nhưng dừng lại **chờ duyệt** trong tab Actions trước khi apply thật.

```yaml
# Thiết kế ban đầu (KHÔNG hoạt động như mong đợi)
on:
  push:
    branches: [main]

jobs:
  terraform-apply:
    environment: production # kỳ vọng: dừng chờ duyệt ở đây
    runs-on: ubuntu-latest
    steps:
      - run: terraform apply -auto-approve
```

#### Vỡ khi áp dụng thực tế

{{% notice warning %}}
📌 **Deploy chạy ngay lập tức, không hề dừng chờ duyệt.** Kiểm tra bằng `gh repo view --json isPrivate` thì thấy lý do: **"Required reviewers"** của GitHub Environment **chỉ khả dụng với repo private khi thuộc Organization có gói GitHub Team/Enterprise trả phí**. Repo `RAGonAWS` là private, thuộc **tài khoản cá nhân**, nên tùy chọn này **không tồn tại trong UI** — không phải cấu hình sai, mà do giới hạn gói dịch vụ.
{{% /notice %}}

Đây là kiểu lỗi khó phát hiện nhất: không có thông báo lỗi, YAML hợp lệ, chỉ **hành vi không như kỳ vọng** — chỉ bắt được khi quan sát Deploy chạy ngay và chủ động tìm nguyên nhân bằng `gh repo view`.

#### Cách sửa: chuyển sang `workflow_dispatch`

```yaml
# .github/workflows/deploy.yml (hiện tại)
on:
  workflow_dispatch: {}

concurrency:
  group: deploy-production
  cancel-in-progress: false

permissions:
  contents: read

jobs:
  terraform-apply:
    name: terraform apply (production)
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

Đổi trigger từ `push: branches: [main]` sang `workflow_dispatch: {}` — **merge vào `main` giờ không tự deploy**; phải **Actions → Deploy → Run workflow** mới apply thật.

Workflow còn đặt `concurrency: group: deploy-production` với `cancel-in-progress: false` để không chạy hai apply cùng lúc lên cùng workspace HCP Terraform.

{{% notice note %}}
`workflow_dispatch` là **"cổng chặn" miễn phí** thay Required reviewers, cùng mục tiêu: **không có cú click chủ động thì không có gì bị apply**. Không phải lúc nào cũng cần đúng tính năng “chuẩn” — cơ chế đơn giản hơn nhưng đạt cùng mục tiêu an toàn vẫn hợp lý khi tính năng chuẩn không khả dụng.
{{% /notice %}}

---

#### Nội dung tiếp theo

- [5.9.3 - Thiết lập thủ công và giới hạn phạm vi](../5.9.3-Manual-Setup-and-Scope-Limitations/)
