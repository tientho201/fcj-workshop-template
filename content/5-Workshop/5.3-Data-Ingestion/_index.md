---
title: "Data Ingestion"
date: 2024-01-01
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

#### Introduction

Stream 1 serves as the ingestion layer for the entire system — converting raw documents uploaded by users (PDFs, scanned images, text files) into structured, semantically searchable data, laying the foundation for Stream 2 (Realtime Q&A) to leverage later.

The stream operates on an event-driven model, consisting of 2 main components:

- **Infrastructure (Terraform)** — S3 stores original documents, SQS acts as an intermediate buffer with 2 levels of Dead Letter Queues, and 3 DynamoDB tables store processing statuses along with vector/BM25 data (completely replacing OpenSearch Serverless from the initial architecture diagram).
- **Processing Logic (Lambda `handler.py`)** — extracts text based on file type, splits documents using a Parent-Child chunking strategy, generates vector embeddings via Amazon Bedrock, and includes a manual OCR confirmation mechanism for scanned documents.

#### Data Flow Diagram

![Detailed Diagram for Stream 1 - Document Processing and Storage](/images/5-Workshop/5.3-Data-Ingestion/image.png)

#### Detailed Contents

The detailed implementation steps of this stream are presented in the sub-pages below:

1. [Infrastructure: S3 and SQS](5.3.1-Infrastructure-S3-SQS/)
2. [Infrastructure: DynamoDB and IAM Permissions](5.3.2-Infrastructure-DynamoDB-IAM/)
3. [Text Extraction by File Type](5.3.3-Text-Extraction/)
4. [Parent-Child Chunking and Embedding](5.3.4-Chunking-Embedding/)
5. [Resume OCR Mechanism and Error Handling](5.3.5-Resume-OCR-Error-Handling/)
6. [End-to-End Testing](5.3.6-End-To-End-Testing/)
