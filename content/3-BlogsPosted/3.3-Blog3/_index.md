---
title: "Blog 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.3. </b> "
---

# From Theory to Execution: The Data Engineering Challenge in a Real-World GenAI/RAG Project

Hello everyone,

In my previous post, I shared my perspective on what a Data Engineer should prioritize learning on AWS to optimize time and be job-ready.

Today, I would like to share in more detail how I applied those AWS mindsets and services directly into my practical project. Although the main problem of the project falls under GenAI / RAG (Retrieval-Augmented Generation), when I started implementing the system architecture, I realized: up to 80% of the system's power and stability lies in Data Engineering.

Below is the full picture of how I applied cloud data processing techniques in this project.

## 1. Data Ingestion & Event-Driven Pipeline

When processing input documents, relying on traditional direct function calls easily creates bottlenecks or overloads the system. Therefore, I built an Event-Driven Architecture:

- **Automatic Trigger (S3 Event Trigger):** Whenever a user or Admin uploads a new document (PDF/TXT/Scan) to Amazon S3 Raw Documents, an S3 Event immediately triggers the processing flow without requiring manual polling.
- **Buffer Queue & System Decoupling (Amazon SQS):** To prevent the system from being overwhelmed when a large volume of files is uploaded simultaneously, data flows through Amazon S3 Event → Amazon SQS (Buffer Queue). Using SQS acts as a load-bearing buffer, combining automatic retries and a Dead Letter Queue (DLQ) to catch failed messages, guaranteeing zero data loss.

## 2. ETL & Unstructured Data Processing

Unstructured text data needs to undergo a strict ETL process before it can serve AI models:

- **Data Extraction (OCR):** AWS Lambda receives messages from SQS and automatically invokes Amazon Textract to perform OCR, accurately extracting text from complex scanned files or PDFs.
- **Structuring & Vector Storage (Chunking & Vectorization):** Processed data is chunked following a Parent-Child model, then sent to the Embedding API for transformation into vectors. All of this structured information is stored in Amazon DynamoDB—optimized for Hybrid Search queries (combining Cosine Similarity and BM25 using the RRF algorithm) to achieve maximum search accuracy.

## 3. Low-Latency Data Retrieval & Caching

A core Data Engineering challenge in Web/App applications is latency and cost:

- **Semantic Caching:** I integrated Amazon ElastiCache Serverless as a smart query caching layer. If a new question has a semantic meaning similar to a previous one, the system immediately returns results from Cache instead of calling expensive LLM models. This technique minimizes latency for end-users (Real-time serving).
- **State Management & Feedback (Transaction Store):** The entire chat history and user feedback store are persisted in DynamoDB—a NoSQL database delivering ultra-fast write speeds with millisecond latency.

## 4. Automated Batch Processing & Continuous Evaluation

Beyond real-time data flows, a standardized data system always requires periodic processing (Batch Pipelines) for quality evaluation:

- **Automated Batch Jobs:** I used Amazon EventBridge Scheduler combined with AWS Lambda to launch the model evaluation pipeline (following the RAGAS evaluation criteria) automatically at fixed daily schedules.
- **Periodic Data Lake Storage:** Daily evaluation results are pushed back into Amazon S3 Evaluation Results acting as a Data Lake for historical logs. This allows the team to easily monitor and analyze system quality trends over time.

## 5. Data Observability & Monitoring

Finally, data running within a Cloud system must be observable to catch issues promptly:

- **Centralized Logs & Metrics:** Amazon CloudWatch collects logs and tracks critical pipeline metrics, such as DLQ Depth, API Gateway 5xx error rates, as well as AI Custom Metrics like Faithfulness, Relevancy, and Precision.
- **Smart Alert Routing:** Incidents are classified by severity (Warning vs Critical). Upon a Critical failure, Amazon SNS works with AWS Chatbot to instantly route alert notifications to Slack or PagerDuty for the engineering team to resolve.

## Three Data Engineering Highlights I Learned From the Project

If you are preparing for a thesis or personal project, here are 3 Data Engineering mindsets I found most valuable:

- **Decoupled Architecture:** The Data Ingestion and Query Serving phases operate independently via SQS queues and DynamoDB. Even if an Admin uploads thousands of PDF files at once, the end-user chat experience remains completely smooth and unaffected.
- **Asynchronous Serverless Processing:** Combining S3 Events + SQS + Lambda makes the system fully asynchronous. Compute resources only spin up when data is flowing through, achieving 100% cloud operation cost optimization.
- **Query Optimization:** Instead of querying LLMs directly in a "naive" manner, combining Semantic Cache (ElastiCache) and Hybrid Search (BM25 + Vector Search) clearly demonstrates a Data Engineer's query performance optimization mindset.

## Conclusion

This project proved one thing to me: being a Data Engineer is not just about writing SQL queries or running Spark scripts, but about the ability to design reliable Cloud infrastructure, automate data flows, and optimize operational costs for the system.

I hope this practical project perspective gives you a clearer picture of how AWS services like S3, SQS, Lambda, DynamoDB, and ElastiCache work together in real-world scenarios.

## Link

<https://www.facebook.com/groups/awsstudygroupfcj/permalink/2240430060055287/#>
