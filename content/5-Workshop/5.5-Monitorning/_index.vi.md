---
title: "Giám sát và cảnh báo hệ thống"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

#### Giới thiệu

Luồng 3 theo dõi sức khỏe Luồng 1 và Luồng 2 theo thời gian thực, phân loại cảnh báo theo mức độ nghiêm trọng, và gửi tới đúng kênh vận hành (email hoặc Slack). Alarm chất lượng thứ năm từ Luồng 4 (`ragas-faithfulness-low`) cũng publish vào cùng SNS topic Critical — không dựng thêm một hệ thống cảnh báo riêng.

Hạ tầng khai báo tại `modules/monitoring/main.tf`, gồm 2 phần chính:

- **Hạ tầng (Terraform)** — 2 SNS topic theo severity (`alerts-info` → email, `alerts-critical` → Slack qua AWS Chatbot), 4 CloudWatch Alarm trong monitoring module (cộng alarm RAGAS thuộc `modules/evaluation`), và 1 CloudWatch Dashboard với 9 widget.
- **Custom metric (ứng dụng tự emit)** — chat-engine emit cache hit/miss qua EMF; evaluation-runner đẩy điểm RAGAS qua `put_metric_data`. Terraform chỉ nối widget/alarm đọc các metric này; không tự tạo chuỗi metric.
  {{% notice note %}}
  📌 Luồng 3 **không phải một sản phẩm giám sát tách rời**. Luồng lắp ráp những gì AWS/Lambda đã tự sinh (metric gốc + log) thành alarm và dashboard có cấu trúc, rồi bổ sung vài custom metric chỉ khi AWS không có sẵn (cache hit rate, điểm RAGAS). `bedrock-throttle` là trường hợp đặc biệt: dùng log metric filter quét `"ThrottlingException"` thay vì metric throttle native của Bedrock.
  {{% /notice %}}

#### Sơ đồ luồng dữ liệu

![Sơ đồ chi tiết Luồng 3 - Monitoring và Alert](/images/5-Workshop/5.5-Monitorning/image.png)

#### Nội dung chi tiết

1. [SNS: 2 kênh theo mức độ nghiêm trọng](5.5.1-SNS-2-Channels-By-Severity/)
2. [CloudWatch Alarms](5.5.2-CloudWatch-Alarms/)
3. [CloudWatch Dashboard và Custom Metrics](5.5.3-Dashboard-Custom-Metrics/)
4. [Kiểm thử end-to-end](5.5.4-End-to-End-Testing/)
