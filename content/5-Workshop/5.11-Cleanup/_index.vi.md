---
title: "Dọn dẹp tài nguyên"
date: 2024-01-01
weight: 11
chapter: false
pre: " <b> 5.11. </b> "
---

{{% notice note %}}
📌 **Đính chính so với suy đoán trước đó:** `scripts/down.sh` **không phải** "tắt tạm giữ nguyên dữ liệu trên AWS" — đây là **destroy thật sự** toàn bộ hạ tầng (đúng mục đích tiết kiệm chi phí ~$2.5/ngày đã nhắc ở [5.9](../5.9-CICD/)), chỉ khác `terraform destroy` trần ở chỗ có thêm các bước dọn dẹp an toàn trước khi destroy, và giữ lại **1 bản backup cục bộ** để `scripts/up.sh` phục hồi dữ liệu khi dựng lại từ đầu.
{{% /notice %}}

#### Vì sao không thể `terraform destroy` trần

Bucket `raw_documents` bật **versioning** (để không mất dữ liệu khi ai đó vô tình ghi đè tài liệu — xem lại [Luồng 1, trang 5.3.1](../5.3-Data-Ingestion/5.3.1-Infrastructure-S3-SQS/)). Hệ quả: xóa object thường (`DELETE`) chỉ tạo ra 1 delete-marker mới, bản thân object cũ **vẫn còn trong bucket** dưới dạng noncurrent version.

{{% notice warning %}}
`terraform destroy` gọi `DeleteBucket`, mà AWS **từ chối xóa bucket còn version** — báo lỗi `BucketNotEmpty` và **dừng giữa chừng**, để lại stack dở dang (một số resource đã xóa, một số chưa). Vì vậy dự án không destroy trực tiếp mà đi qua `scripts/down.sh`.
{{% /notice %}}

#### `down.sh` làm gì, theo đúng thứ tự

1. **Đọc output hiện tại** — `terraform output -raw raw_documents_bucket_name` / `evaluation_results_bucket_name`. Nếu không đọc được (stack đã destroy từ trước), thoát luôn, không làm gì thêm.
2. **Sao lưu tài liệu** — `aws s3 sync` toàn bộ `raw_documents` bucket về `./backup-s3-documents/` (thư mục cục bộ, gitignore, không lên git).
3. **Dọn sạch cả 2 bucket** (`raw_documents` **và** `evaluation_results`) — không chỉ `aws s3 rm --recursive` (chỉ xóa object hiện tại), mà còn **lặp xóa theo trang cả `Versions` lẫn `DeleteMarkers`** qua `aws s3api list-object-versions` + `delete-objects` (giới hạn 900 khóa/lần vì API `delete-objects` nhận tối đa 1000).
4. **`terraform destroy -auto-approve`** — lúc này cả 2 bucket đã rỗng thật, không còn vướng `BucketNotEmpty`.

```bash
#!/usr/bin/env bash
# scripts/down.sh (Mã nguồn thực tế 4 bước)

RAW_BUCKET=$(terraform output -raw raw_documents_bucket_name 2>/dev/null) || exit 0
EVAL_BUCKET=$(terraform output -raw evaluation_results_bucket_name 2>/dev/null || echo "")

# 1. Sao lưu tài liệu về thư mục cục bộ
mkdir -p "$BACKUP_DIR"
aws s3 sync "s3://$RAW_BUCKET/" "$BACKUP_DIR/" --only-show-errors

# 2. Hàm dọn sạch bucket kể cả Versions và DeleteMarkers
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

# 3. Dọn sạch 2 bucket S3
empty_bucket "$RAW_BUCKET"
empty_bucket "$EVAL_BUCKET"

# 4. Destroy toàn bộ stack Terraform
terraform destroy -auto-approve
```

#### Cái gì KHÔNG bị xóa (có chủ đích)

| Thành phần                              | Vì sao giữ lại                                                                                                                                                                 |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Budget alert `rag-app-monthly-cost`** | Nằm ngoài phạm vi Terraform stack chính (tạo 1 lần, không đổi), nên vẫn tiếp tục theo dõi chi phí kể cả khi hạ tầng đã tắt                                                     |
| **Tài liệu ở `backup-s3-documents/`**   | Không phải "sót lại" — đây chính là mục đích của bước 2: `scripts/up.sh` sẽ tự động `aws s3 sync` ngược lại từ thư mục này khi dựng lại stack, không cần upload tay lại từ đầu |

#### Rủi ro đã phòng ngừa: ECR repository của Luồng 4

`aws_ecr_repository.evaluation_runner` (`modules/evaluation/main.tf`) dùng để chứa Docker image của Lambda `evaluation-runner` (Luồng 4 — RAGAS, xem lại [5.6.1](../5.6-RAGAS/5.6.1-EventBridge-Lambda-Container/)).

{{% notice warning %}}
📌 **Bug thật được tìm ra qua rà soát code tĩnh, chưa từng gây sự cố thật:** ECR mặc định **từ chối xóa một repository còn image bên trong**. Nếu image đã từng được push, `terraform destroy` sẽ fail đúng ở resource này (`RepositoryNotEmptyException`) — trong khi `scripts/down.sh` **chỉ dọn tay 2 bucket S3** trước khi destroy, **không có bước tương đương cho ECR**. Kết quả nếu không sửa: 2 bucket destroy trót lọt, nhưng ECR repo bị bỏ lại — stack dở dang, vẫn phát sinh phí lưu trữ image dù người dùng tưởng đã tắt sạch.
{{% /notice %}}

**Đã sửa** bằng cách thêm `force_delete = true` vào resource:

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

Với `force_delete = true`, `terraform destroy` **tự xóa sạch mọi image trong repo** trước khi xóa chính repo đó — không cần thao tác thủ công (`aws ecr batch-delete-image`) như trước.

{{% notice note %}}
Tại thời điểm phát hiện, `evaluation_image_pushed` vẫn đang là `false` (Luồng 4 chưa từng deploy thật trên hạ tầng — xem lại [5.6.4](../5.6-RAGAS/5.6.4-Testing-Deployment-Notes/)), nên đây là lỗi được tìm ra và vá qua **rà soát code tĩnh**, chưa từng thực sự gây sự cố trên hệ thống — nhưng nếu không sửa, **chắc chắn sẽ vỡ ngay lần đầu tiên** ai đó bật Luồng 4 rồi chạy `down.sh`. Đây là ví dụ tốt cho việc rà soát chủ động (proactive review) phát hiện được lỗi trước khi nó có cơ hội xảy ra thật — đáng đưa vào báo cáo như một minh chứng cho tư duy phòng ngừa, không chỉ sửa lỗi khi đã xảy ra.
{{% /notice %}}

#### Xác minh đã dọn sạch

```bash
terraform state list          # rỗng hoặc lỗi "no state" nghĩa là đã destroy xong
aws s3 ls                     # không còn bucket rag-app-dev-* nào
aws ecr describe-repositories # không còn repo rag-app-dev-evaluation-runner
```
