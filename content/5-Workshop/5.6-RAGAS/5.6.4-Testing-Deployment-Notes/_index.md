---
title: "Testing and Production Deployment Notes"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.6.4 </b> "
---

#### First-time Deployment Workflow (2 Phases)

Because of the gate mechanism via `evaluation_image_pushed` presented on [page 5.6.1](../5.6.1-EventBridge-Lambda-Container/), the deployment process for Flow 4 **is very different from the previous 3 flows** — it cannot be deployed by running `terraform apply` just once:

1. **Apply Phase 1** (`evaluation_image_pushed = false`) → creates only the ECR repo + Lambda IAM Role.
2. **Build Docker image** locally, ensuring `ragas`, `langchain-aws`, `datasets`, `pandas` are installed.
3. **Push image** to the ECR repo created in step 1.
4. **Enable variable** `evaluation_image_pushed = true` in `.tfvars`.
5. **Apply Phase 2** → Lambda, EventBridge Schedule, scheduler IAM Role, and RAGAS Alarm are actually created.

{{% notice tip %}}
If you need to **update code** in `evaluation_runner.py` later (not first-time deployment), simply build/push the new image to ECR and run `aws lambda update-function-code --function-name rag-dev-evaluation-runner --image-uri <ecr-uri>:latest` — **no need** to toggle `evaluation_image_pushed` or rerun Terraform, as the Lambda resource already exists.
{{% /notice %}}

#### Test scenarios

| #   | Scenario                                                            | Expected result                                                                                                         |
| --- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| 1   | Run `terraform apply` phase 1 without pushing an image              | Only ECR repo + IAM Role are created, no errors                                                                         |
| 2   | Build and push Docker image to ECR                                  | Image appears with `latest` tag on ECR Console                                                                          |
| 3   | Set `evaluation_image_pushed = true`, apply phase 2                 | Lambda, Schedule, and RAGAS Alarm created successfully                                                                  |
| 4   | Manually invoke `evaluation_runner` (do not wait for 2 AM schedule) | Completes without runtime errors (even if RAGAS scores are not fully accurate yet since it is a skeleton)               |
| 5   | Check S3 `evaluation_results`                                       | File `evaluation/<date>/results.json` appears with detailed data                                                        |
| 6   | Check CloudWatch namespace `RAGEvaluation`                          | 4 metrics (Faithfulness, AnswerRelevancy, ContextPrecision, ContextRecall) have values                                  |
| 7   | Check "RAGAS Evaluation Scores" widget on Flow 3 Dashboard          | Data is displayed (no longer empty as described on [page 5.5.3](../../5.5-Monitorning/5.5.3-Dashboard-Custom-Metrics/)) |
| 8   | Simulate low Faithfulness score (test data)                         | Alarm `ragas-faithfulness-low` transitions to `ALARM`, message appears on Slack via `alerts-critical` channel           |

#### Known Limitations

After bridging the gap joining the 2 tables via GSI ([5.6.2](../5.6.2-IAM-Alarm-RAGAS/), [5.6.3](../5.6.3-RAGAS-Evaluation-Logic/)), 3 real limitations still remain:

| Limitation                                     | Details                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **TTL mismatch between 2 tables**              | `feedback` has no TTL (persists indefinitely), while `chat_history` has a default TTL of **30 days**. Feedback whose original question turn has been deleted by TTL can no longer be joined (as mentioned in [5.6.3](../5.6.3-RAGAS-Evaluation-Logic/)) — the job must run daily to avoid this.               |
| **RAGAS↔Bedrock wiring is still illustrative** | The original skeleton explicitly states that specific `ragas`/`langchain` versions should be pinned and re-tested with real data before trusting the scores — this part **has not been verified with real data**.                                                                                             |
| **Not yet deployed**                           | `evaluation_image_pushed = false` currently — this Lambda (including EventBridge Schedule, alarm `ragas-faithfulness-low`) **does not actually exist on the infrastructure yet**, requiring building + pushing the Docker image then enabling the flag (see [5.6.1](../5.6.1-EventBridge-Lambda-Container/)). |

{{% notice warning %}}
Since `evaluation_runner.py` is a **skeleton implementation**:

- **Flows 1, 2, 3** — actually executed, tested end-to-end, can be presented with actual metrics/logs obtained.
- **Flow 4** — architectural design (Terraform infrastructure, IAM, data flow, alarms) is complete and sound, but **the execution logic of RAGAS inside Lambda remains illustrative/skeleton frame**, requiring further refinement (especially the `Scan` vs `Query` issue on `chat_history` mentioned on [page 5.6.3](../5.6.3-RAGAS-Evaluation-Logic/)) before being considered production-ready.

{{% /notice %}}
