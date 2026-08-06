---
title: "RAG Evaluation with RAGAS"
date: 2024-01-01
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

#### Introduction

Stream 4 automatically measures and evaluates chatbot response quality on a daily basis using the **RAGAS** framework. Infrastructure is declared in `modules/evaluation/main.tf`.

{{% notice warning %}}
📌 **This is a skeleton/placeholder implementation** — unlike the previous 3 streams which are fully operational and tested end-to-end.
{{% /notice %}}

The stream consists of 3 main parts:

- **Infrastructure (Terraform)** — EventBridge Scheduler invokes the Lambda directly every day (without going through SQS/S3), the Lambda is packaged as a container image (unlike the other 2 zip-packaged Lambdas), and a **2-phase gate** mechanism via the `evaluation_image_pushed` variable — the most critical deployment trap in this stream.
- **IAM and Quality Alarm** — read permissions for `feedback`/`chat_history`, dedicated S3 bucket write permissions, and the `ragas-faithfulness-low` alarm alerting when the chatbot shows signs of hallucination.
- **Evaluation Logic (`evaluation_runner.py`)** — fetches yesterday's data, runs 4 RAGAS metrics using Bedrock as the evaluator, writes detailed results to S3, and publishes average scores to CloudWatch.

#### Data Flow Diagram

![Diagram of Stream 4 - RAG Evaluation RAGAS](/images/5-Workshop/5.6-RAGAS/image.png)
_Diagram: Daily EventBridge Scheduler trigger → RAGAS Evaluation Runner Lambda (reading data from DynamoDB Feedback Store and Chat History) → storing results in S3 Evaluation Results._

#### Detailed Contents

1. [Infrastructure: EventBridge Scheduler and Lambda Container Image](5.6.1-EventBridge-Lambda-Container/)
2. [IAM Permissions and RAG Quality Alarm](5.6.2-IAM-Alarm-RAGAS/)
3. [RAGAS Evaluation Logic (evaluation_runner.py)](5.6.3-RAGAS-Evaluation-Logic/)
4. [Testing and Production Deployment Notes](5.6.4-Testing-Deployment-Notes/)
