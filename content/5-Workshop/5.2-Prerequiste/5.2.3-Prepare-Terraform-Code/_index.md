---
title: "Prepare Terraform Code"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.2.3 </b> "
---

### Tools to install

Before you start, make sure the following tools are installed:

**Terraform** version **1.5** or later to deploy infrastructure as code.

**AWS CLI** to interact with AWS services and perform testing steps from the command line.

**Python** version **3.12** or later to develop the Lambda functions.

You can check that the tools were installed successfully by running the following commands:

```
terraform version
aws --version
python3 --version
```

![](/images/5-Workshop/5.2-Prerequisite/image5.2.3a.png)

### Configure AWS credentials environment variables for the CLI

In the terminal window, configure AWS credentials so that the AWS CLI can authenticate and run commands throughout the deployment and testing process:

```
aws configure
```

Enter the following information in order:

```
AWS Access Key ID       [None]: <ACCESS_KEY_ID>
AWS Secret Access Key   [None]: <SECRET_ACCESS_KEY>
Default region name     [None]: us-east-1
Default output format   [None]:
```

After configuring, restrict access to the credentials file so only the current account can read it:

```
chmod 600 ~/.aws/credentials
```

Verify the credentials again to make sure authentication succeeded:

```
aws sts get-caller-identity
```

### Configure the Terraform foundation

In the project's root directory, create a new folder named terraform. This folder contains the Terraform files used to deploy the AWS infrastructure.

Create a **providers.tf** file to declare the Terraform version, configure the connection to HCP Terraform, and set up the AWS Provider. Replace **YOUR-ORGANIZATION-NAME** with your Organization name and **YOUR-WORKSPACE-NAME** with the Workspace name created on HCP Terraform.

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

### Connect and initialize resources

In the terminal, navigate to the folder containing the Terraform files, then run the command below to sign in and connect the Terraform CLI to HCP Terraform:

```
terraform login
```

![Connect and initialize resources](/images/5-Workshop/5.2-Prerequisite/image5.2.3b.png)

- Open the URL shown in the terminal window:

![Connect and initialize resources](/images/5-Workshop/5.2-Prerequisite/image5.2.3c.png)

- Enter a Description if you want, to make the token easy to identify.
- Choose an expiration time for the API Token.
- Click Generate token.

![Connect and initialize resources](/images/5-Workshop/5.2-Prerequisite/image5.2.3d.png)

- Copy the newly created Token into the Terminal:

![Connect and initialize resources](/images/5-Workshop/5.2-Prerequisite/image5.2.3e.png)

- After signing in successfully, initialize Terraform to download the providers and connect to the Workspace:

```
terraform init
```

![Connect and initialize resources](/images/5-Workshop/5.2-Prerequisite/image5.2.3f.png)

After completing the steps above, the working environment is fully configured and ready to deploy the **RAG** system in the following sections.

### Repo directory structure

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

### Prepare Git and configure .gitignore

Run `git init`, create a `.gitignore` excluding `.terraform/`, `*.tfstate`, `venv/`, `__pycache__/`, and push to the remote repo (GitHub) agreed upon by the team. This helps protect sensitive information and keeps the repository clean.

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

- The `.terraform/` folder: This is where Terraform caches the providers downloaded when running `terraform init`. This folder is large and can be easily recreated, so it doesn't need to be committed to the repository.

- The `*.tfstate`, `*.tfstate.*` files: These are infrastructure state files, which may contain sensitive information. Since the project uses HCP Terraform to manage remote state, the state is already stored centrally on HCP, so there's no need to keep it locally in the repository.

- The `*.tfvars` files: These hold real variable values such as the alert notification email, so they aren't committed to the repository to avoid leaking information. We only keep the sample file `*.tfvars.example`.

#### Next content

- [Data Ingestion](../../5.3-Data-Ingestion/)
