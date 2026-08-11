---
title: "End-to-End Testing"
date: 2024-01-01
weight: 8
chapter: false
pre: " <b> 5.4.8 </b> "
---

After completing the infrastructure ([5.4.1](../5.4.1-API-Gateway-Cognito),[5.4.2](../5.4.2-Cache-Guardrails-IAM/)) and all processing logic ([5.4.3](../5.4.3-Cache-Lookup-Query-Rewriting/) → [5.4.7](../5.4.7-Alternative-Route/)), the final step is to test real-world scenarios for Flow 2.

#### Test scenarios

| #   | Scenario                                                              | Expected result                                                                                                |
| --- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 1   | First question of a new session, not in cache yet                     | Cache miss → runs full pipeline (retrieval → generation) → saves to cache + history                            |
| 2   | Ask the exact same question again (in a different new session)        | Cache hit → responds almost instantly, no Bedrock call logs observed                                           |
| 3   | Ask subsequent question within the same session (with history)        | Bypasses cache (`cacheable = False`), runs Query Rewriting using Haiku                                         |
| 4   | Context-dependent question (e.g., "what about the second one?")       | Rewritten into a standalone question before retrieval                                                          |
| 5   | Question out of scope of ingested documents                           | Returns "no relevant documents found", does not call Claude to generate answer                                 |
| 6   | Question containing policy-violating content (input Guardrail test)   | Blocked before reaching retrieval                                                                              |
| 7   | Simulate Bedrock returning `ThrottlingException`                      | Client receives HTTP 429 with `retryable: true`, ERROR log appears on CloudWatch                               |
| 8   | Confirm OCR via `/documents-decision` for pending document            | Receives 202 immediately, `document_processor` invoked asynchronously, `/status` updates progress              |
| 9   | Click 👍 under a normal answer                                        | Button locks + highlights, new item appears in `feedback` table with correct `message_id`                      |
| 10  | Click 👎 under an answer retrieved from cache or blocked by Guardrail | Still works correctly — confirms `message_id` is generated for both branches (see [5.4.7](../5.4.7-Feedback/)) |
| 11  | Click feedback during simulated network disconnection                 | Button automatically unlocks after error, clickable again, not stuck in "sending" state                        |

##### Figure 1: Test 1 result

![Test 1 result](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-1.png)

##### Figure 2: Test 2 result

![Test 2 result](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-2.png)

##### Figure 3: Test 3 result

![Test 3 result](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-3.png)

##### Figure 4: Test 4 result

![Test 4 result](/images/5-Workshop/5.4-Realtime-QA/image5.4.8-4.png)

#### Outcomes

- API secured with Cognito, serving all 4 routes via a single Lambda, reducing the number of functions to manage.
- Exact-match cache significantly reduces cost/latency for verbatim repeated questions at the start of a session, while preventing the risk of returning incorrect cached responses for context-dependent questions.
- Query Rewriting using Claude Haiku makes retrieval more accurate for follow-up questions in conversations, with a safe fallback when rewriting fails.
- Custom Hybrid Search implemented on DynamoDB (cosine + BM25 → RRF) completely replaces OpenSearch, removing 1 infrastructure component from operation.
- 2-layer Guardrail (input/output) and separate Throttling handling ensure the system is both content-safe and client-friendly during overload.
- `/documents-decision` route connects seamlessly with Flow 1, completing the human-in-the-loop experience for OCR confirmation.
