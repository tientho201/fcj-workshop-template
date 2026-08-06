---
title: "Backend"
date: 2024-01-01
weight: 8
chapter: false
pre: " <b> 5.8. </b> "
---

#### Backend Code Architecture

This section describes design decisions **at the code level**, distinct from the "Processing Flow" chapter (5.3 → 5.6 — described from an end-to-end infrastructure pipeline perspective). Both main Lambdas (`document_processor`, `chat_engine`) follow a core principle throughout:

{{% notice note %}}
📌 **Zero external dependency** — except for exactly 1 library (`pypdf`, vendored directly into git rather than installed during `terraform apply`), all logic relies solely on `boto3`/`botocore` and standard Python libraries. Even the Redis client is a ~70-line **custom-written** RESP client, instead of using `redis-py`.
{{% /notice %}}

As a result of this principle, 2 shared libraries (`bm25.py`, `vector_store.py`) exist as **identical copies** in both Lambdas — without using Lambda Layers or a shared package, keeping each Lambda **completely self-contained**, trading off manual synchronization when making modifications.

#### Detailed Contents

1. [Retrieval Algorithms](5.8.1-Retrieval-Algorithms/)
2. [Semantic Cache & Observability](5.8.2-Cache-and-Observability/)
3. [Error Handling & Security](5.8.3-Error-Handling-and-Security/)
4. [Backend Testing](5.8.4-Backend-Testing/)
