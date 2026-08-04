---
title: "Cấu hình HCP Terraform"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2.2 </b> "
---

### Tạo Organization cho dự án

**Bước 2.1 — Tạo tài khoản và Organization**

Truy cập [https://app.terraform.io/app](https://app.terraform.io/app) và đăng nhập. Nếu chưa có tài khoản, bạn hãy chọn Sign up để tạo tài khoản mới. Tại giao diện chính, chọn Create organization để bắt đầu tạo một tổ chức cho dự án (ví dụ `RAG-on-AWS`).

![Tạo Organization trên HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-1a.png)

![Tạo Organization trên HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-1b.png)

**Bước 2.2 — Tạo Workspace**
 
Tạo **Workspace** mới, chọn workflow **CLI-driven** (Terraform chạy local, HCP Terraform chỉ lưu state) hoặc **Version control workflow** (tự động chạy khi push code lên Git) tùy nhóm quyết định. Ở đây tôi sẽ chọn **CLI-Driven**. Sau đó nhấn cho **Create** để tạo.

![Tạo Workspace trên HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-2a.png)

![Tạo Workspace trên HCP Terraform](/images/5-Workshop/5.2-Prerequisite/image5.2.2-2b.png)

**Bước 2.3 — Cấu hình biến môi trường AWS credentials**
 
Trong Workspace vừa tạo, vào **Variables**, thêm 2 biến môi trường (Environment variables, đánh dấu **Sensitive**): `AWS_ACCESS_KEY_ID` và `AWS_SECRET_ACCESS_KEY` của IAM User đã tạo ở phần trước.

![Thêm AWS credentials vào Workspace Variables](/images/5-Workshop/5.2-Prerequisite/image5.2.2-3a.png)

![Thêm AWS credentials vào Workspace Variables](/images/5-Workshop/5.2-Prerequisite/image5.2.2-3b.png)

**Bước 2.4 — Kết nối Terraform CLI local với HCP Terraform**
 
Chạy `terraform login` để xác thực token, khai báo block `cloud` trong file cấu hình Terraform:
 
```hcl
terraform {
  cloud {
    organization = "RAG-on-AWS"
    workspaces {
      name = "RAG-app"
    }
  }
}
```
 
Chạy `terraform init` để xác nhận kết nối thành công tới HCP Terraform.

Chạy lệnh này `terraform apply` để bắt đầu lần chạy đầu tiên cho không gian làm việc này.

#### Nội dung tiếp theo

- [Chuẩn bị Code Terrform](../5.2.3-Prepare-Terraform-Code/_index.vi.md)