---
title: "Xây dựng CI/CD"
date: 2026-08-09
weight: 9
chapter: false
pre: " <b> 5.9. </b> "
---

#### Bối cảnh — vì sao thiết kế thế này

Hạ tầng chạy trên **HCP Terraform** (workspace `RAGonAWS/RAG-app`, execution mode **"Remote"**) — nghĩa là `terraform plan`/`apply` thật sự chạy trên worker của HashiCorp, **không phải** trên máy cục bộ hay trên GitHub Actions runner. CI/CD ở đây vì vậy **không tự chạy `terraform apply`** như một pipeline build-deploy thông thường, mà chỉ **kích hoạt** các lệnh Terraform CLI, xác thực bằng token (`TF_API_TOKEN`), rồi để HCP Terraform làm phần việc nặng.

{{% notice warning %}}
📌 **Ràng buộc chi phí quan trọng nhất chi phối toàn bộ thiết kế:** hạ tầng được **chủ động tắt** bằng `scripts/down.sh` khi không dùng để tiết kiệm (~$2.5/ngày lúc bật) — nên **merge code không được phép tự động apply**, kẻo vô tình dựng lại toàn bộ stack đang tắt.
{{% /notice %}}

2 workflow trong `.github/workflows/`, chạy trên GitHub Actions:

- **`ci.yml`** — chạy trên mọi PR và mọi push vào `main`, gồm **5 job** (4 chạy song song; `terraform-plan` chờ `terraform-checks` và chỉ chạy trên PR). Không đổi gì trên AWS.
- **`deploy.yml`** — kích hoạt **thủ công** (`workflow_dispatch`), là nơi thực sự chạy `terraform apply`.

#### Nội dung chi tiết

1. [ci.yml — Kiểm tra trên mọi PR](5.9.1-CI-Workflow/)
2. [deploy.yml — Vấn đề Required reviewer và cách sửa](5.9.2-Deploy-Workflow/)
3. [Thiết lập thủ công và giới hạn phạm vi](5.9.3-Manual-Setup-and-Scope-Limitations/)
