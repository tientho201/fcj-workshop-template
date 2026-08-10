---
title: "Thử nghiệm hệ thống"
date: 2024-01-01
weight: 10
chapter: false
pre: " <b> 5.10. </b> "
---

{{% notice note %}}
📌 **Đây là trang tổng hợp**, không lặp lại chi tiết đã có ở từng phần — mỗi dòng trong bảng dưới đây trỏ thẳng tới nơi trình bày đầy đủ nhất. Nếu cần một mục "Testing" độc lập cho báo cáo, đây chính là trang nên trích dẫn: nó cho cái nhìn toàn cảnh mà không cần đọc lại 8 trang riêng lẻ.
{{% /notice %}}

#### Tổng hợp: mỗi phần đã test gì, bằng cách nào

| Phần                  | Đã test gì                                                                                    | Cách test                           | Xem chi tiết                                             |
| --------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------------------------- |
| Luồng 1 — Ingestion   | 4 loại file (text/PDF text/PDF scan/ảnh), xác nhận OCR, resume/cancel                         | Thủ công qua UI                     | [5.3.6](../5.3-Data-Ingestion/5.3.6-End-To-End-Testing/) |
| Luồng 2 — Realtime QA | 11 kịch bản: cache, rewrite, retrieval, guardrail, throttle, feedback                         | Thủ công qua UI + DevTools + script | [5.4.8](../5.4-Realtime-QA/5.4.8-End-To-End-Testing/)    |
| Luồng 3 — Monitoring  | 8 kịch bản: 4 alarm 2 chiều (ALARM↔OK), subscription, Slack OAuth                             | Thủ công, giả lập lỗi               | [5.5.4](../5.5-Monitorning/5.5.4-End-to-End-Testing/)    |
| Luồng 4 — RAGAS       | Kiến trúc dữ liệu (GSI, IAM) đã đúng; **logic RAGAS chưa test với dữ liệu thật, chưa deploy** | Chưa test được (chưa deploy)        | [5.6.4](../5.6-RAGAS/5.6.4-Testing-Deployment-Notes/)    |
| Frontend              | 11 kịch bản: đăng nhập, upload, OCR dialog, responsive                                        | Thủ công qua trình duyệt            | [5.7.4](../5.7-Frontend/5.7.4-Test-end-to-end/)          |
| Backend (logic thuần) | **33 unit test** (chunking, BM25, vector store, RRF, drift 4 file dùng chung)                 | **Tự động**, chạy trong CI          | [5.8.4](../5.8-Backend/5.8.4-Backend-Testing/)           |
| CI — kiểm tra tĩnh    | Lint (`ruff`), cú pháp Terraform (`fmt`/`validate`), quét secret (`gitleaks`)                 | **Tự động**, mọi PR/push            | [5.9.1](../5.9-CICD/5.9.1-CI-Workflow/)                  |
| Kiểm thử thủ công/E2E | 3 loại PDF dựng tay, script gọi API thật, **1 bug thật bắt được** (504 → thiếu VPC endpoint)  | Thủ công, không lặp lại tự động     | [5.10.1](5.10.1-Manual-E2E-Testing/)                     |

#### Đọc nhanh: 2 loại kiểm thử khác bản chất

Nhìn xuyên suốt bảng trên, toàn bộ việc kiểm thử của dự án rơi vào đúng 2 loại — cần phân biệt rõ khi trình bày báo cáo, không nên gộp chung thành "đã test đầy đủ":

|                                    | Tự động (lặp lại được)                                         | Thủ công (không lặp lại được)                                                                                  |
| ---------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Gồm những gì**                   | Lint/validate/secret-scan (CI) + 33 unit test (pytest)         | Toàn bộ 11+8+11+4 kịch bản còn lại trong bảng trên, cộng câu chuyện E2E ở [5.10.1](5.10.1-Manual-E2E-Testing/) |
| **Chạy lại mỗi lần push?**         | ✅                                                             | ❌                                                                                                             |
| **Kiểm tra logic nghiệp vụ thật?** | Một phần — chỉ phần logic thuần Python, không chạm `boto3`/AWS | ✅ — chạy trên hạ tầng AWS thật                                                                                |

{{% notice warning %}}
**Khoảng trống thật còn lại** (không phải suy đoán, đã xác nhận ở [5.8.4](../5.8-Backend/5.8.4-Backend-Testing/)): `handler.py` của cả 2 Lambda — phần **gọi trực tiếp `boto3`/AWS** — chưa có test tự động ở bất kỳ tầng nào, chỉ được kiểm chứng qua kiểm thử thủ công (không lặp lại được). Route `/feedback` và join qua GSI `message_id-index` cũng cùng tình trạng này. Đây là điểm nên đưa thẳng vào mục "Hạn chế" của báo cáo.
{{% /notice %}}

#### Nội dung chi tiết (không trùng lặp với các phần trên)

1. [Kiểm thử thủ công/E2E](5.10.1-Manual-E2E-Testing/)
