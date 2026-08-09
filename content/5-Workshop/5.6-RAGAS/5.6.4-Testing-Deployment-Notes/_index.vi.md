---
title: "Testing và lưu ý triển khai"
date: 2024-01-01
weight: 4
chapter: false
pre: " <b> 5.6.4 </b> "
---

#### Quy trình triển khai lần đầu (2 pha)

Vì cơ chế gate qua `evaluation_image_pushed` đã trình bày ở [trang 5.6.1](../5.6.1-EventBridge-Lambda-Container/), quy trình triển khai Luồng 4 **khác hẳn 3 luồng trước** — không thể chỉ chạy `terraform apply` một lần:

1. **Apply lần 1** (`evaluation_image_pushed = false`) → chỉ tạo ECR repo + IAM Role Lambda.
2. **Build Docker image** local, đảm bảo cài đủ `ragas`, `langchain-aws`, `datasets`, `pandas`.
3. **Push image** lên ECR vừa tạo ở bước 1.
4. **Bật biến** `evaluation_image_pushed = true` trong `.tfvars`.
5. **Apply lần 2** → Lambda, EventBridge Schedule, IAM Role scheduler, Alarm RAGAS mới thực sự được tạo.

{{% notice tip %}}
Nếu sau này cần **cập nhật code** `evaluation_runner.py` (không phải lần đầu triển khai), chỉ cần build/push image mới lên ECR rồi chạy `aws lambda update-function-code --function-name rag-dev-evaluation-runner --image-uri <ecr-uri>:latest` — **không cần** quay lại đổi biến `evaluation_image_pushed` hay chạy lại Terraform, vì resource Lambda đã tồn tại từ trước.
{{% /notice %}}

#### Kịch bản test

| #   | Kịch bản                                                      | Kỳ vọng                                                                                                                         |
| --- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Chạy `terraform apply` lần 1 khi chưa push image              | Chỉ thấy ECR repo + IAM Role được tạo, không có lỗi                                                                             |
| 2   | Build và push Docker image lên ECR                            | Image xuất hiện với tag `latest` trên ECR Console                                                                               |
| 3   | Bật `evaluation_image_pushed = true`, apply lần 2             | Lambda, Schedule, Alarm RAGAS được tạo thành công                                                                               |
| 4   | Invoke thủ công `evaluation_runner` (không chờ lịch 2h sáng)  | Chạy xong, không lỗi runtime (dù kết quả RAGAS có thể chưa hoàn toàn chính xác vì là skeleton)                                  |
| 5   | Kiểm tra S3 `evaluation_results`                              | File `evaluation/<ngày>/results.json` xuất hiện với dữ liệu chi tiết                                                            |
| 6   | Kiểm tra CloudWatch namespace `RAGEvaluation`                 | 4 metric (Faithfulness, AnswerRelevancy, ContextPrecision, ContextRecall) có giá trị                                            |
| 7   | Kiểm tra widget "RAGAS Evaluation Scores" ở Dashboard Luồng 3 | Đã có dữ liệu hiển thị (không còn trống như mô tả ở [trang 5.5.3](../../5.5-Flow-3-Monitoring/5.5.3-Dashboard-Custom-Metrics/)) |
| 8   | Giả lập điểm Faithfulness thấp (dữ liệu test)                 | Alarm `ragas-faithfulness-low` chuyển `ALARM`, tin nhắn xuất hiện trên Slack qua kênh `alerts-critical`                         |

![Kết quả invoke thủ công evaluation_runner lần đầu](../images/08-manual-invoke-first-run.png)
_Ảnh minh họa: log CloudWatch của lần invoke thủ công đầu tiên, S3 và metric CloudWatch đã có dữ liệu._

#### Giới hạn còn tồn tại (nêu rõ để báo cáo trung thực)

Sau khi khép kín gap nối 2 bảng qua GSI ([5.6.2](../5.6.2-IAM-Alarm-RAGAS/), [5.6.3](../5.6.3-Logic-Danh-gia-RAGAS/)), vẫn còn 3 giới hạn thật cần nêu rõ:

| Giới hạn                                          | Chi tiết                                                                                                                                                                                                                                                                     |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Lệch TTL giữa 2 bảng**                          | `feedback` không có TTL (tồn tại vĩnh viễn), `chat_history` có TTL mặc định **30 ngày**. Feedback mà lượt hỏi gốc đã bị TTL xóa thì không nối được nữa (đã nói ở [5.6.3](../5.6.3-Logic-Danh-gia-RAGAS/)) — job phải chạy đều hàng ngày mới tránh được.                      |
| **Phần wiring RAGAS↔Bedrock vẫn ở dạng minh họa** | Tác giả gốc của skeleton này tự ghi rõ nên chốt phiên bản `ragas`/`langchain` cụ thể và test lại với dữ liệu thật trước khi tin tưởng điểm số — phần này **chưa được kiểm chứng bằng dữ liệu thật**.                                                                         |
| **Chưa deploy được**                              | `evaluation_image_pushed = false` hiện tại — Lambda này (kể cả EventBridge Schedule, alarm `ragas-faithfulness-low`) **chưa tồn tại thật trên hạ tầng**, cần build + push image Docker rồi bật cờ mới có hiệu lực (xem lại [5.6.1](../5.6.1-EventBridge-Lambda-Container/)). |

{{% notice tip %}}
So với lần ghi nhận trước, phạm vi "chưa chắc chắn" của Luồng 4 đã **thu hẹp đáng kể**: phần kiến trúc dữ liệu (2 bảng, GSI, IAM) giờ đã đúng và có thể tin tưởng; phần còn lại chưa chắc chắn chỉ còn ở tầng **thực thi** (chưa deploy, chưa test với dữ liệu thật). Đây là cách trình bày tiến độ trung thực và rõ ràng cho báo cáo — không phải "vẫn y nguyên chưa làm gì" mà là "đã sửa xong phần thiết kế, còn lại phần vận hành thực tế".
{{% /notice %}}

{{% notice warning %}}
Vì `evaluation_runner.py` là **bản skeleton**, khi trình bày trong báo cáo/luận văn, nên phân biệt rõ ràng với 3 luồng trước:

- **Luồng 1, 2, 3** — đã chạy thật, đã test end-to-end, có thể trình bày kèm số liệu/log thực tế thu được.
- **Luồng 4** — thiết kế kiến trúc (hạ tầng Terraform, IAM, luồng dữ liệu, alarm) đã hoàn chỉnh và đúng đắn, nhưng **phần thực thi logic RAGAS bên trong Lambda còn ở dạng minh họa/khung sườn**, cần tiếp tục hoàn thiện (đặc biệt là vấn đề `Scan` thay vì `Query` trên `chat_history` đã nói ở [trang 5.6.3](../5.6.3-Logic-Danh-gia-RAGAS/)) trước khi coi là sẵn sàng cho production.

Trình bày trung thực điểm này giúp báo cáo có độ tin cậy cao hơn, thay vì để người chấm tự phát hiện qua việc hỏi sâu hoặc xem code thực tế.
{{% /notice %}}

#### Checklist hoàn thành Luồng 4

- [ ] Hiểu rõ quy trình apply 2 pha, không nhầm lẫn "apply xong" với "luồng đã hoạt động"
- [ ] Docker image build/push thành công lên ECR
- [ ] Lambda, EventBridge Schedule, IAM Role, Alarm đã được tạo sau apply lần 2
- [ ] IAM Role `evaluation_runner` chỉ có đúng 4 nhóm quyền, hiểu rõ vì sao `PutMetricData` phải dùng `Resource: "*"` kèm Condition
- [ ] Alarm `ragas-faithfulness-low` trỏ đúng về kênh `alerts-critical` đã có sẵn từ Luồng 3
- [ ] Test invoke thủ công thành công, có dữ liệu ở cả S3 và CloudWatch
- [ ] Đã ghi chú rõ giới hạn "skeleton" của luồng này nếu đưa vào báo cáo/luận văn
