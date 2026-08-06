---
title: "Giám sát và cảnh báo hệ thống"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

#### Giới thiệu

Luồng 3 giám sát toàn bộ hệ thống theo thời gian thực, phân loại cảnh báo theo mức độ nghiêm trọng, và gửi thông báo tới đúng kênh vận hành. Hạ tầng được khai báo tại `modules/monitoring/main.tf`.

{{% notice note %}}
📌 Luồng 3 **không phải một hệ thống giám sát tách rời**, mà là lắp ráp lại đúng những gì AWS/Lambda đã tự sinh ra (metric gốc + log) thành alarm/dashboard có cấu trúc, cộng thêm vài custom metric ứng dụng tự log khi cần thứ AWS không có sẵn (cache hit rate, RAGAS score).
{{% /notice %}}

Luồng gồm 3 phần chính:

- **2 kênh SNS theo mức độ nghiêm trọng** — `alerts-info` (Warning, gửi email) và `alerts-critical` (Critical, route sang Slack qua AWS Chatbot).
- **4 CloudWatch Alarm** — mỗi alarm nối đúng 1 trong 2 kênh trên, trong đó `bedrock-throttle` là alarm đặc biệt không dùng metric gốc mà quét log.
- **1 CloudWatch Dashboard với 8 widget** — 6 widget đọc metric có sẵn của AWS, 2 widget cuối đọc custom metric do Lambda tự log ra (EMF).

#### Sơ đồ luồng dữ liệu

![Sơ đồ chi tiết Luồng 3 - Monitoring và Alert](/images/5-Workshop/5.5-Monitorning/image.png)

#### Nội dung chi tiết

1. [SNS: 2 kênh theo mức độ nghiêm trọng](5.5.1-SNS-2-Channels-By-Severity/)
2. [CloudWatch Alarms](5.5.2-CloudWatch-Alarms/)
3. [CloudWatch Dashboard và Custom Metrics](5.5.3-Dashboard-Custom-Metrics/)
4. [Kiểm thử end-to-end](5.5.4-End-to-End-Testing/)
