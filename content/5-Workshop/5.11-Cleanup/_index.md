---
title: "Clean Up Resources"
date: 2024-01-01
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---

{{% notice note %}}
📌 **Correction regarding previous assumption:** `scripts/down.sh` is **not** a "temporary pause while preserving AWS data" — it is a **true destruction** of the entire infrastructure (intended to save the ~$2.5/day cost mentioned in [5.9](../5.9-CICD/)). The only difference from a raw `terraform destroy` is that it includes safety cleanup steps before destroying, and retains **1 local backup** so `scripts/up.sh` can restore data when rebuilding from scratch.
{{% /notice %}}

#### Why Raw `terraform destroy` Fails

The `raw_documents` bucket has **versioning enabled** (to prevent data loss if someone accidentally overwrites a document — see [Stream 1, Page 5.3.1](../5.3-Data-Ingestion/5.3.1-Infrastructure-S3-SQS/)). As a result, deleting a normal object (`DELETE`) only creates a new delete-marker, while the old object **remains in the bucket** as a noncurrent version.

{{% notice warning %}}
`terraform destroy` invokes `DeleteBucket`, but AWS **refuses to delete a bucket containing object versions** — reporting a `BucketNotEmpty` error and **halting midway**, leaving a partial stack (some resources deleted, others remaining). Therefore, the project does not destroy directly, but executes via `scripts/down.sh`.
{{% /notice %}}

#### What `down.sh` Does, Step by Step

1. **Read current outputs** — `terraform output -raw raw_documents_bucket_name` / `evaluation_results_bucket_name`. If unreadable (stack already destroyed), exit immediately without further action.
2. **Back up documents** — `aws s3 sync` the entire `raw_documents` bucket to `./backup-s3-documents/` (a local directory, gitignored, not pushed to git).
3. **Empty both buckets** (`raw_documents` **and** `evaluation_results`) — not just `aws s3 rm --recursive` (which only removes current objects), but **paginated deletion of both `Versions` and `DeleteMarkers`** via `aws s3api list-object-versions` + `delete-objects` (limited to 900 keys per batch because the `delete-objects` API accepts a maximum of 1000).
4. **`terraform destroy -auto-approve`** — at this point both buckets are truly empty, bypassing `BucketNotEmpty`.

```bash
#!/usr/bin/env bash
# scripts/down.sh (Actual 4-step source code)

RAW_BUCKET=$(terraform output -raw raw_documents_bucket_name 2>/dev/null) || exit 0
EVAL_BUCKET=$(terraform output -raw evaluation_results_bucket_name 2>/dev/null || echo "")

# 1. Back up documents to local directory
mkdir -p "$BACKUP_DIR"
aws s3 sync "s3://$RAW_BUCKET/" "$BACKUP_DIR/" --only-show-errors

# 2. Function to empty bucket including Versions and DeleteMarkers
empty_bucket() {
  local bucket="$1"
  [ -z "$bucket" ] && return 0
  aws s3 rm "s3://$bucket" --recursive --only-show-errors 2>/dev/null || true

  local kind
  for kind in Versions DeleteMarkers; do
    while true; do
      aws s3api list-object-versions --bucket "$bucket" --max-items 900 \
        --query "{Objects: ${kind}[].{Key:Key,VersionId:VersionId}}" \
        --output json > "$TMP_DIR/del.json" 2>/dev/null || break
      grep -q '"Objects": null' "$TMP_DIR/del.json" && break
      aws s3api delete-objects --bucket "$bucket" \
        --delete "file://$TMP_DIR/del.json" >/dev/null 2>&1 || break
    done
  done
}

# 3. Empty both S3 buckets
empty_bucket "$RAW_BUCKET"
empty_bucket "$EVAL_BUCKET"

# 4. Destroy the entire Terraform stack
terraform destroy -auto-approve
```

#### What Is Intentionally NOT Deleted

| Component                               | Reason Retained                                                                                                                                                                                 |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Budget alert `rag-app-monthly-cost`** | Defined outside the main Terraform stack scope (created once, unchanged), so cost monitoring continues even when infrastructure is shut down                                                    |
| **Documents in `backup-s3-documents/`** | Not leftover data — this is the explicit goal of step 2: `scripts/up.sh` automatically performs an `aws s3 sync` back from this directory when rebuilding the stack, avoiding manual re-uploads |

#### Prevented Risk: Stream 4 ECR Repository

`aws_ecr_repository.evaluation_runner` (`modules/evaluation/main.tf`) is used to store Docker images for the `evaluation-runner` Lambda (Stream 4 — RAGAS, see [5.6.1](../5.6-RAGAS/5.6.1-EventBridge-Lambda-Container/)).

{{% notice warning %}}
📌 **Real bug discovered via static code review (never caused a production outage):** By default, ECR **refuses to delete a repository containing images**. If an image was ever pushed, `terraform destroy` fails on this specific resource (`RepositoryNotEmptyException`) — whereas `scripts/down.sh` **only manually empties the 2 S3 buckets** before destroying, with **no equivalent step for ECR**. Result if unfixed: 2 buckets destroyed successfully, but ECR repo left behind — leaving a partial stack that continues incurring image storage fees while users assume everything is shut down.
{{% /notice %}}

**Fixed** by adding `force_delete = true` to the resource:

```hcl
resource "aws_ecr_repository" "evaluation_runner" {
  name                 = "${local.name_prefix}-evaluation-runner"
  image_tag_mutability = "MUTABLE"

  # Without this, `terraform destroy` fails on this resource specifically
  # once an image has ever been pushed (RepositoryNotEmptyException) —
  # scripts/down.sh empties both S3 buckets by hand before destroying but
  # has no equivalent step for ECR, so this repo would otherwise be the one
  # thing left standing after an otherwise-successful down.sh run.
  force_delete = true

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

With `force_delete = true`, `terraform destroy` **automatically deletes all images inside the repo** before deleting the repository itself — eliminating the need for manual intervention (`aws ecr batch-delete-image`).

{{% notice note %}}
At the time of discovery, `evaluation_image_pushed` was still set to `false` (Stream 4 had never been deployed on real infrastructure — see [5.6.4](../5.6-RAGAS/5.6.4-Testing-Deployment-Notes/)), so this flaw was identified and patched via **static code review** prior to causing an actual operational failure — but without this fix, **it would inevitably break the very first time** someone enabled Stream 4 and ran `down.sh`. This serves as an excellent example of proactive code review catching bugs before they manifest, making it a valuable highlight for technical reports.
{{% /notice %}}

#### Verification of Clean Destroy

```bash
terraform state list          # Empty or "no state" error indicates successful destroy
aws s3 ls                     # No remaining rag-app-dev-* buckets
aws ecr describe-repositories # No remaining rag-app-dev-evaluation-runner repo
```
