---
title: "Thiết lập thủ công và giới hạn phạm vi"
date: 2026-08-09
weight: 3
chapter: false
pre: " <b> 5.9.3. </b> "
---

#### Cần thiết lập một lần (trong GitHub UI, không tự động hóa qua code)

CI/CD chỉ lo phần **trigger** — thiết lập ban đầu vẫn cần thao tác thủ công trong GitHub UI:

1. **Thêm secret `TF_API_TOKEN`** — `Settings → Secrets and variables → Actions → New repository secret`. Token lấy từ HCP Terraform: `User Settings → Tokens` (hoặc **Team Token** của org `RAGonAWS` nếu muốn giới hạn quyền hơn). Token này **dùng chung cho cả 2 workflow** (`ci.yml` cho `terraform plan`, `deploy.yml` cho `terraform apply`).

![Thêm secret TF_API_TOKEN trong GitHub repository settings](../images/03-github-secret-tf-api-token.png)
*Settings → Secrets and variables → Actions, secret `TF_API_TOKEN` đã thêm.*

{{% notice note %}}
📌 **Không cần thêm AWS access key nào vào GitHub.** `terraform apply` chạy trên **worker của HCP Terraform** (execution mode Remote — xem [trang tổng quan](../)), không phải trên GitHub Actions runner. AWS credentials nằm ở **workspace variables trong HCP Terraform**, đã cấu hình sẵn, tách biệt khỏi GitHub Actions. Đây là lợi ích phụ của Remote execution: bề mặt tấn công phía GitHub nhỏ hơn so với đặt AWS credentials trực tiếp làm GitHub Secret.
{{% /notice %}}

#### Không nằm trong phạm vi CI/CD hiện tại

{{% notice warning %}}
Các điểm sau **cố tình chưa tự động hóa** — ghi rõ như giới hạn phạm vi, không phải quên:

- **`scripts/up.sh` / `scripts/down.sh`** (tạo lại tài khoản Cognito, nạp backup tài liệu khi bật/tắt hạ tầng theo ràng buộc chi phí — xem [trang tổng quan](../)) **vẫn chạy thủ công**. CI/CD chỉ lo `terraform apply` hạ tầng, không lo khởi tạo lại dữ liệu/tài khoản sau khi dựng lại stack.
- **`handler.py` của cả hai Lambda** (đường `boto3` / Bedrock / Cognito / S3 / SQS thật) **chưa** nằm trong unit suite. `python-test` cover **33** test trên module thuần + chống drift bản sao (`chunking`, `bm25`, `vector_store`, `retrieval` với DynamoDB fake, đồng bộ `tracing`/`embeddings`) — xem [5.9.1](../5.9.1-CI-Workflow/) và [5.10.3](../../5.10-System-Testing/5.10.3-Layer-3-Automated-Unit-Testing/).
{{% /notice %}}

#### Tổng kết luồng CI/CD

```
PR mở/cập nhật → ci.yml
                 (secret-scan, python-lint, python-test, terraform-checks
                  song song; terraform-plan sau checks, chỉ trên PR)
               → review code

Merge vào main → KHÔNG tự deploy (đã bỏ trigger push)

Deploy thật    → Actions → "Deploy" → "Run workflow"
               → deploy.yml → terraform apply -auto-approve
               (chạy trên HCP Terraform worker; AWS creds từ
                workspace variables)
```

{{% notice tip %}}
Bản mô tả dài hơn còn có trong `docs/CI-CD.md` của repo ứng dụng — hữu ích khi bổ sung ảnh/số liệu thật. Nếu doc đó lệch thời gian (ví dụ trước khi thêm `python-test`), lấy `.github/workflows/` làm nguồn đúng.
{{% /notice %}}

#### Checklist hoàn thành phần CI/CD

- [ ] Hiểu vì sao `deploy.yml` không tự chạy khi merge (ràng buộc chi phí / `down.sh`, không phải lỗi thiết kế)
- [ ] Hiểu Required reviewer không khả dụng trên private repo cá nhân, và cách `workflow_dispatch` thay thế
- [ ] Đã thêm secret `TF_API_TOKEN`; xác nhận `ci.yml` chạy được `terraform plan` trên PR thử
- [ ] Đã chạy `deploy.yml` bằng “Run workflow” ít nhất 1 lần; apply thành công
- [ ] Đã ghi nhận giới hạn phạm vi: `up.sh`/`down.sh` thủ công; biết `python-test` = 33 test logic thuần (không phải `handler.py`); gap trỏ về 5.10.3
