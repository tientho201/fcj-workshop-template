---
title: "Các bước chuẩn bị"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---


Trước khi triển khai bất kỳ luồng nào, cần chuẩn bị đầy đủ tài khoản, quyền truy cập và công cụ dưới đây. Bỏ qua bước chuẩn bị thường là nguyên nhân gây tắc nghẽn giữa chừng — đặc biệt là việc xin quyền truy cập model trên Bedrock, vốn có thể mất thời gian chờ duyệt.
 
#### Yêu cầu chung
 
Trước khi bắt đầu, bạn cần chuẩn bị các điều kiện sau:
 
* Một tài khoản AWS.
* Region sử dụng trong dự án này là **N. Virginia (us-east-1)** — khu vực hỗ trợ đầy đủ các model cần dùng trên Amazon Bedrock (Claude 3, Titan Embeddings).
* Một tài khoản **HCP Terraform** (app.terraform.io) để quản lý remote state.
* Quyền truy cập model (Model access) đã được cấp cho Claude 3 và Titan Embeddings trên Amazon Bedrock.
#### Công cụ cần cài đặt
 
* **Terraform** (phiên bản 1.5 trở lên) để triển khai hạ tầng dưới dạng Infrastructure as Code.
* **AWS CLI** (v2) để thao tác và kiểm thử với các dịch vụ AWS từ dòng lệnh.
* **Python** (phiên bản 3.12) để phát triển các hàm AWS Lambda (Document Processor, Chat Engine, RAGAS Evaluation Runner).
* **Git** để quản lý mã nguồn và Terraform state theo version.
* Một trình soạn thảo mã nguồn như **Visual Studio Code**.
Sau khi cài đặt xong, kiểm tra nhanh bằng các lệnh sau:
 
```bash
aws --version
terraform -version
python3 --version
git --version
```


#### IAM permissions

Tài khoản/IAM User dùng để triển khai dự án (chạy `terraform apply`) cần được gắn policy với các nhóm quyền sau, tương ứng với từng nhóm dịch vụ trong kiến trúc:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ComputeLambda",
            "Effect": "Allow",
            "Action": [
                "lambda:CreateFunction",
                "lambda:DeleteFunction",
                "lambda:GetFunction",
                "lambda:UpdateFunctionCode",
                "lambda:UpdateFunctionConfiguration",
                "lambda:InvokeFunction",
                "lambda:AddPermission",
                "lambda:RemovePermission",
                "lambda:CreateEventSourceMapping",
                "lambda:DeleteEventSourceMapping",
                "lambda:GetEventSourceMapping",
                "lambda:ListEventSourceMappings",
                "lambda:PublishLayerVersion",
                "lambda:GetLayerVersion",
                "lambda:DeleteLayerVersion",
                "lambda:TagResource",
                "lambda:ListTags"
            ],
            "Resource": "*"
        },
        {
            "Sid": "StorageS3",
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:DeleteBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket",
                "s3:GetBucketVersioning",
                "s3:PutBucketVersioning",
                "s3:PutBucketPolicy",
                "s3:GetBucketPolicy",
                "s3:PutBucketPublicAccessBlock",
                "s3:GetBucketPublicAccessBlock",
                "s3:PutEncryptionConfiguration",
                "s3:PutBucketNotification",
                "s3:GetBucketNotification",
                "s3:PutLifecycleConfiguration"
            ],
            "Resource": "*"
        },
        {
            "Sid": "MessagingQueueAndEvents",
            "Effect": "Allow",
            "Action": [
                "sqs:CreateQueue",
                "sqs:DeleteQueue",
                "sqs:GetQueueAttributes",
                "sqs:SetQueueAttributes",
                "sqs:SendMessage",
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:TagQueue",
                "sqs:ListQueues",
                "sns:CreateTopic",
                "sns:DeleteTopic",
                "sns:Subscribe",
                "sns:Unsubscribe",
                "sns:Publish",
                "sns:ListTopics",
                "sns:SetTopicAttributes",
                "sns:GetTopicAttributes",
                "events:PutRule",
                "events:DeleteRule",
                "events:PutTargets",
                "events:RemoveTargets",
                "events:DescribeRule",
                "events:ListRules",
                "events:ListTargetsByRule"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DatabaseAndCache",
            "Effect": "Allow",
            "Action": [
                "dynamodb:CreateTable",
                "dynamodb:DeleteTable",
                "dynamodb:DescribeTable",
                "dynamodb:UpdateTable",
                "dynamodb:PutItem",
                "dynamodb:GetItem",
                "dynamodb:Query",
                "dynamodb:Scan",
                "dynamodb:UpdateItem",
                "dynamodb:DeleteItem",
                "dynamodb:TagResource",
                "elasticache:CreateServerlessCache",
                "elasticache:DeleteServerlessCache",
                "elasticache:DescribeServerlessCaches",
                "elasticache:ModifyServerlessCache",
                "elasticache:CreateCacheSubnetGroup",
                "elasticache:DeleteCacheSubnetGroup",
                "elasticache:DescribeCacheSubnetGroups",
                "elasticache:ListTagsForResource"
            ],
            "Resource": "*"
        },
        {
            "Sid": "SearchAndGenAI",
            "Effect": "Allow",
            "Action": [
                "aoss:CreateCollection",
                "aoss:DeleteCollection",
                "aoss:BatchGetCollection",
                "aoss:ListCollections",
                "aoss:CreateSecurityPolicy",
                "aoss:GetSecurityPolicy",
                "aoss:UpdateSecurityPolicy",
                "aoss:ListSecurityPolicies",
                "aoss:CreateAccessPolicy",
                "aoss:GetAccessPolicy",
                "aoss:UpdateAccessPolicy",
                "aoss:ListAccessPolicies",
                "aoss:APIAccessAll",
                "aoss:TagResource",
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream",
                "bedrock:GetFoundationModel",
                "bedrock:ListFoundationModels",
                "bedrock:CreateGuardrail",
                "bedrock:GetGuardrail",
                "bedrock:UpdateGuardrail",
                "bedrock:DeleteGuardrail",
                "bedrock:ListGuardrails",
                "bedrock:ApplyGuardrail",
                "textract:DetectDocumentText",
                "textract:AnalyzeDocument",
                "textract:StartDocumentTextDetection",
                "textract:GetDocumentTextDetection"
            ],
            "Resource": "*"
        },
        {
            "Sid": "APIAndAuth",
            "Effect": "Allow",
            "Action": [
                "apigateway:GET",
                "apigateway:POST",
                "apigateway:PUT",
                "apigateway:PATCH",
                "apigateway:DELETE",
                "cognito-idp:CreateUserPool",
                "cognito-idp:DeleteUserPool",
                "cognito-idp:UpdateUserPool",
                "cognito-idp:DescribeUserPool",
                "cognito-idp:CreateUserPoolClient",
                "cognito-idp:DeleteUserPoolClient",
                "cognito-idp:UpdateUserPoolClient",
                "cognito-idp:DescribeUserPoolClient",
                "cognito-idp:ListUserPools",
                "cognito-idp:AdminCreateUser",
                "cognito-idp:AdminSetUserPassword",
                "cognito-idp:TagResource"
            ],
            "Resource": "*"
        },
        {
            "Sid": "MonitoringAndLogs",
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData",
                "cloudwatch:GetMetricData",
                "cloudwatch:PutDashboard",
                "cloudwatch:GetDashboard",
                "cloudwatch:DeleteDashboards",
                "cloudwatch:PutMetricAlarm",
                "cloudwatch:DeleteAlarms",
                "cloudwatch:DescribeAlarms",
                "cloudwatch:DescribeAlarmsForMetric",
                "logs:CreateLogGroup",
                "logs:DeleteLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogGroups",
                "logs:PutRetentionPolicy",
                "chatbot:CreateSlackChannelConfiguration",
                "chatbot:DeleteSlackChannelConfiguration",
                "chatbot:DescribeSlackChannelConfigurations",
                "chatbot:UpdateSlackChannelConfiguration"
            ],
            "Resource": "*"
        },
        {
            "Sid": "IAMForServiceRoles",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:GetRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PutRolePolicy",
                "iam:GetRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:CreatePolicy",
                "iam:DeletePolicy",
                "iam:GetPolicy",
                "iam:ListPolicyVersions",
                "iam:ListRoles",
                "iam:PassRole",
                "iam:TagRole"
            ],
            "Resource": "*"
        }
    ]
}

```

#### Nội dung

1. [Cấu hình AWS Credentials](5.2.1-Configure-AWS-Credentials/_index.vi.md)
2. [Cấu hình HCP Terraform](5.2.2-Configure-HCP-Terraform/_index.vi.md)
2. [Chuẩn bị Code Terraform](5.2.3-Prepare-Terraform-Code/_index.vi.md)