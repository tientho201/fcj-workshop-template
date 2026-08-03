---
title: "Week 4 Worklog"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4 Objectives:

- Master Serverless fundamentals (Lambda, SQS, IAM Role) and independently implement Flow 1: Document Processing & Storage.

### Tasks to be carried out this week:

| Day | Task                                                                                                                                           | Start Date | Completion Date | Reference Material                                                                                                               |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 2   | - Learn about AWS Lambda: function, trigger, layers, environment variables, cold start, IAM execution role                                     | 07/13/2025 | 07/13/2025      | [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/)                                                                  |
| 3   | - Learn about Amazon S3 (Event Notification) and Amazon SQS: Standard Queue, Dead Letter Queue (DLQ), retry policy                             | 07/14/2025 | 07/14/2025      | [Amazon S3 Documentation](https://docs.aws.amazon.com/AmazonS3/)<br>[Amazon SQS Documentation](https://docs.aws.amazon.com/sqs/) |
| 4   | - Learn about Amazon Textract (OCR scanned files/images)<br>- **Practice:** Lambda polling messages from SQS and processing files              | 07/15/2025 | 07/15/2025      | [Amazon Textract Documentation](https://docs.aws.amazon.com/textract/)                                                           |
| 5   | - Learn about advanced IAM Roles: least privilege, resource-based policy between services (S3 → SQS → Lambda)                                  | 07/16/2025 | 07/16/2025      | [AWS IAM Documentation](https://docs.aws.amazon.com/IAM/)                                                                        |
| 6   | - **Practice:** Complete build of Flow 1 – S3 (upload) → S3 Event → SQS (buffer + retry) → Lambda (Document Processor) → OCR for scanned files | 07/17/2025 | 07/17/2025      | Personal Project                                                                                                                 |

### Week 4 Achievements:

- Understood and implemented basic event-driven architecture: S3 Event → SQS → Lambda.
- Configured Dead Letter Queue (DLQ) and retry mechanisms for failed document processing.
- Automated OCR processing for images/scanned documents using Amazon Textract.
- Configured secure least privilege permissions between services using IAM Roles.
- Established an automated document ingestion pipeline, ready for vector generation in the following week.
