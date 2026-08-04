---
title: "Prerequisites"
date: 2024-01-01
weight: 2
chapter: false
pre: " <b> 5.2. </b> "
---


Before deploying any workflow, you need to fully prepare the accounts, access, and tools below. Skipping the preparation step is often the cause of mid-way blockers — especially requesting model access on Bedrock, which can take time to be approved.

#### General requirements

Before you start, you need to prepare the following:

* An AWS account.
* The region used in this project is **N. Virginia (us-east-1)** — the region that fully supports the required models on Amazon Bedrock (Claude 3, Titan Embeddings).
* An **HCP Terraform** account (app.terraform.io) to manage remote state.
* Model access has been granted for Claude 3 and Titan Embeddings on Amazon Bedrock.
#### Tools to install

* **Terraform** (version 1.5 or later) to deploy infrastructure as code.
* **AWS CLI** (v2) to operate and test AWS services from the command line.
* **Python** (version 3.12) to develop AWS Lambda functions (Document Processor, Chat Engine, RAGAS Evaluation Runner).
* **Git** to manage source code and version the Terraform state.
* A code editor such as **Visual Studio Code**.
After installation, quickly verify with the following commands:

```bash
aws --version
terraform -version
python3 --version
git --version
```


#### IAM permissions

The account/IAM User used to deploy the project (running `terraform apply`) needs to have a policy attached with the following permission groups, corresponding to each service group in the architecture:

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

#### Contents

1. [Configure AWS Credentials](5.2.1-Configure-AWS-Credentials/_index.md)
2. [Configure HCP Terraform](5.2.2-Configure-HCP-Terraform/_index.md)
2. [Prepare Terraform Code](5.2.3-Prepare-Terraform-Code/_index.md)