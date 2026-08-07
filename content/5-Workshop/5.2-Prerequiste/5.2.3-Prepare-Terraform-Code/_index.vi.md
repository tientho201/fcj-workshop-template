---
title: "Chuẩn bị Code Terraform"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.2.3 </b> "
---

### Công cụ cần cài đặt

Trước khi bắt đầu, hãy bảo đảm rằng các công cụ sau đã được cài đặt:

**Terraform** phiên bản **1.5** trở lên để triển khai hạ tầng theo mô hình Infrastructure as Code.

**AWS CLI** để tương tác với các dịch vụ AWS và thực hiện các bước kiểm thử từ dòng lệnh.

**Python** phiên bản **3.12** trở lên để phát triển các hàm Lambda.

Bạn có thể kiểm tra các công cụ đã được cài đặt thành công bằng cách chạy các lệnh sau:

```
terraform version
aws --version
python3 --version
```

![](/images/5-Workshop/5.2-Prerequisite/image5.2.3a.png)

### Cấu hình biến môi trường AWS credentials cho CLI

Trên cửa sổ terminal, cấu hình AWS credentials để AWS CLI có thể xác thực và thực thi các lệnh trong suốt quá trình triển khai cũng như kiểm thử hệ thống:

```
aws configure
```

Lần lượt nhập các thông tin sau:

```
AWS Access Key ID       [None]: <ACCESS_KEY_ID>
AWS Secret Access Key   [None]: <SECRET_ACCESS_KEY>
Default region name     [None]: us-east-1
Default output format   [None]:
```

Sau khi cấu hình xong, hãy giới hạn quyền truy cập vào file credentials để chỉ tài khoản hiện tại có thể đọc được:

```
chmod 600 ~/.aws/credentials
```

Kiểm tra lại credentials để bảo đảm việc xác thực đã thành công:

```
aws sts get-caller-identity
```

### Cấu hình nền tảng Terraform

Tại thư mục gốc của dự án, tạo thư mục mới có tên là terraform. Thư mục này chứa các file Terraform dùng để triển khai hạ tầng AWS.

Tạo file **providers.tf** để khai báo phiên bản Terraform, cấu hình kết nối với HCP Terraform và thiết lập AWS Provider. Thay **TEN-TO-CHUC** bằng tên Organization của bạn và **TEN-WORKSPACE** bằng tên Workspace đã tạo trên HCP Terraform.

```terraform
terraform {
  required_version = ">= 1.5"

  cloud {
    organization = "RAGonAWS"

    workspaces {
      name = "RAG-app"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
    null = {
      source  = "hashicorp/null"
      version = "~> 3.2"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

### Kết nối và khởi tạo tài nguyên

Trong terminal, chuyển đến thư mục chứa các file Terraform, sau đó chạy lệnh dưới đây để đăng nhập và kết nối Terraform CLI với HCP Terraform:

```
terraform login
```

![Kết nối và khởi tạo tài nguyên](/images/5-Workshop/5.2-Prerequisite/image5.2.3b.png)

- Mở đường dẫn được hiển thị trên cửa sổ terminal:

![Kết nối và khởi tạo tài nguyên](/images/5-Workshop/5.2-Prerequisite/image5.2.3c.png)

- Nhập Description nếu muốn để dễ dàng nhận biết token.
- Chọn thời gian hết hạn cho API Token.
- Nhấn Generate token.

![Kết nối và khởi tạo tài nguyên](/images/5-Workshop/5.2-Prerequisite/image5.2.3d.png)

- Copy Token vừa tạo vào Terminal:

![Kết nối và khởi tạo tài nguyên](/images/5-Workshop/5.2-Prerequisite/image5.2.3e.png)

- Sau khi đăng nhập thành công, khởi tạo Terraform để tải tài provider và kết nối Workspace:

```
terraform init
```

![Kết nối và khởi tạo tài nguyên](/images/5-Workshop/5.2-Prerequisite/image5.2.3f.png)

Sau khi hoàn thành các bước trên, môi trường làm việc đã được cấu hình đầy đủ và sẵn sàng để triển khai hệ thống **RAG** trong các phần tiếp theo.

### Cấu trúc thư mục repo

Tổ chức repo theo từng luồng để dễ phân công và review:

```
Project/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── bootstrap/                    # backend Terraform (S3+DynamoDB state) - trước khi có HCP Terraform
│   ├── main.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── variables.tf
├── docker/
│   └── evaluation-runner/        # image chạy RAGAS evaluation trên ECS/Lambda container
│       ├── Dockerfile
│       ├── evaluation_runner.py
│       └── requirements.txt
├── modules/                       # Terraform modules chính
│   ├── evaluation/                # ECR repo + S3 bucket cho kết quả RAGAS
│   ├── ingestion/                 # S3 → SQS → Lambda document_processor
│   │   └── lambda_src/document_processor/
│   │       ├── bm25.py
│   │       ├── chunking.py
│   │       ├── embeddings.py
│   │       ├── handler.py
│   │       ├── requirements.txt    # pypdf vendor (không cài lúc apply)
│   │       ├── tracing.py
│   │       └── vector_store.py
│   ├── monitoring/                 # CloudWatch dashboard, alarms, SNS, Chatbot
│   ├── networking/                 # VPC + VPC endpoints (chỉ để chat_engine nối ElastiCache)
│   └── query/                      # API Gateway + Cognito + Lambda chat_engine
│       └── lambda_src/chat_engine/
│           ├── bm25.py
│           ├── cache.py
│           ├── embeddings.py
│           ├── handler.py
│           ├── retrieval.py        # hybrid search: cosine + BM25 → RRF
│           ├── tracing.py
│           └── vector_store.py
├── scripts/
│   ├── down.sh                     # backup + destroy toàn bộ stack
│   └── up.sh                       # apply + tạo lại Cognito + nạp backup
├── ui/
│   ├── README.md
│   └── index.html                   # UI single-page, không build step
├── main.tf                          # root module — ghép các module lại
├── outputs.tf
├── providers.tf                     # cloud { } block (HCP Terraform workspace)
├── ruff.toml
├── terraform.tfvars.example
└── variables.tf
```

### Chuẩn bị Git và cấu hình .gitignore

Chạy `git init`, tạo `.gitignore` loại trừ `.terraform/`, `*.tfstate`, `venv/`, `__pycache__/`, push lên remote repo (GitHub) đã thống nhất trong nhóm. Việc này giúp bảo vệ thông tin nhạy cảm và giữ cho repository gọn gàng.

```gitignore
# AWS credentials — NEVER commit. Standard filename AWS IAM console uses
# when exporting an access key CSV. If you see one of these in the working
# tree, treat the key as compromised the moment it might have been staged —
# rotate it in IAM regardless of whether it actually got pushed.
*accessKeys*.csv
*_accessKeys*.csv
.aws/
credentials
*.pem
*.ppk

# Terraform
**/.terraform/*
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfvars
*.tfvars.json
!terraform.tfvars.example
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc
# .terraform.lock.hcl is intentionally NOT ignored — commit it (pins
# provider versions/checksums for reproducible `terraform init`, same
# reasoning as committing a package-lock.json).

# Packaged Lambda artifacts (generated by archive_file data sources)
modules/*/build/

# Docker
docker/**/__pycache__/
**/__pycache__/
*.pyc
```

- Thư mục `.terraform/`: Đây là nơi Terraform lưu cache các provider được tải về khi chạy `terraform init`. Thư mục này có dung lượng lớn và có thể tái tạo dễ dàng nên không cần đưa lên repository.

- Các file `*.tfstate`, `*.tfstate.*`: Đây là file trạng thái hạ tầng, có thể chứa thông tin nhạy cảm. Vì dự án sử dụng HCP Terraform để quản lý remote state, trạng thái đã được lưu tập trung trên HCP nên không cần lưu cục bộ trong repository.

- Các file `*.tfvars`: Đây là nơi chứa giá trị biến thật như email nhận cảnh báo, nên không đưa lên repository để tránh lộ thông tin. Chúng ta chỉ giữ lại file mẫu `*.tfvars.example`.
