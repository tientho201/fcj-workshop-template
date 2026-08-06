---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Building an Intelligent Document Q&A System with RAG Architecture on AWS

#### Overview

**RAG (Retrieval-Augmented Generation)** is an architecture combining information retrieval with large language models (LLMs), ensuring generated responses strictly adhere to actual ground-truth data rather than relying solely on the pre-trained knowledge of the model.

In this workshop, we will build a complete **RAG Knowledge Assistant** system on the AWS Serverless platform. The system allows users to upload documents (PDF/TXT/scanned images), automatically digitize and semantically index them, and subsequently submit queries to receive answers generated directly from the uploaded content — featuring content moderation, semantic caching to optimize costs, and an automated monitoring and response quality evaluation mechanism.

The entire system is divided into four main processing flows, corresponding to four independent yet closely coupled functional groups:

- **Flow 1 — Data Ingestion**: Collect user documents, perform OCR processing for scanned files/images, convert content into vector embeddings, and store them in a semantic vector search store.
- **Flow 2 — Realtime Q&A**: Receive queries, retrieve relevant context, generate answers via LLM, with a caching mechanism to minimize costs and response latency.
- **Flow 3 — Monitoring & Alert**: Monitor the entire system in real-time, classify alerts by severity level, and dispatch notifications to operational channels.
- **Flow 4 — RAG Evaluation**: Automatically measure and evaluate answer quality on a daily schedule.

#### Content

1. [System Architecture Overview](5.1-Workshop-overview/)
2. [Prerequisites & AWS Account Setup](5.2-Prerequisites/)
3. [Flow 1 - Document Processing and Storage](5.3-Data-Ingestion/)
4. [Flow 2 - Realtime Q&A with Semantic Cache](5.4-Realtime-QA/)
5. [Flow 3 - System Monitoring and Alerting](5.5-Monitoring/)
6. [Flow 4 - RAG Quality Evaluation with RAGAS](5.6-RAGAS/)
7. [Frontend](5.7-Frontend/)
8. [Backend](5.8-Backend/)
9. [Building CI/CD](5.9-CICD/)
10. [System Testing](5.10-System-Testing/)
11. [Clean up Resources](5.11-Cleanup/)
