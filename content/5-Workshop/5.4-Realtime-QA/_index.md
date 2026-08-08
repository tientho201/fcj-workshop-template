---
title: "Realtime QA with Semantic Cache"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.4. </b> "
---

#### Introduction

Stream 2 handles incoming user questions, retrieves relevant context from data indexed in Stream 1, generates answers via Amazon Bedrock, and returns responses to users — complete with content moderation, caching to reduce costs, and history storage for Stream 4 quality evaluation.

The stream operates with **a single `chat_engine` Lambda serving 4 API routes**, consisting of 2 main parts:

- **Infrastructure (Terraform)** — API Gateway with 4 routes sharing 1 Lambda, Cognito for user authentication, ElastiCache Serverless for caching, Bedrock Guardrails for content moderation, and least-privilege IAM Roles.
- **Processing Logic (Lambda `handler.py`)** — branches based on route, with the primary `_handle_chat` route running sequentially: cache lookup → query rewriting → input guardrail → hybrid search (DynamoDB) → answer generation → output guardrail → write cache/history.
  {{% notice note %}}
  📌 Important difference from the initial naming: **ElastiCache here is an exact-match cache** (question hash → answer, with TTL), **not a true semantic cache** — because ElastiCache Serverless Redis does not support the RediSearch/vector search module. The `cache_similarity_threshold` variable is currently reserved for future upgrades (MemoryDB or self-managed Redis+RediSearch).
  {{% /notice %}}

#### Data Flow Diagram

![Detailed Diagram for Stream 2 - Realtime Q&A](/images/5-Workshop/5.4-Realtime-QA/image.png)

#### Detailed Contents

1. [Infrastructure: API Gateway and Cognito](5.4.1-API-Gateway-Cognito/)
2. [Infrastructure: Semantic Cache, Guardrails and IAM](5.4.2-Cache-Guardrails-IAM/)
3. [Cache Lookup and Query Rewriting](5.4.3-Cache-Lookup-Query-Rewriting/)
4. [Hybrid Search and Retrieval](5.4.4-Hybrid-Search-Retrieval/)
5. [Answer Generation and History Storage](5.4.5-Answer-Generation-History-Storage/)
6. [Error Handling and OCR Decision Integration](5.4.6-Error-Handling-OCR-Decision/)
7. [Alternative Route](5.4.7-Alternative-Route/)
8. [End-to-End Testing](5.4.8-End-To-End-Testing/)
