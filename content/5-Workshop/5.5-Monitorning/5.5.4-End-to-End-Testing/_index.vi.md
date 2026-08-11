---
title: "Kiểm thử end-to-end"
date: 2026-08-03
weight: 4
chapter: false
pre: " <b> 5.5.4. </b> "
---

Sau khi hoàn tất SNS ([5.5.1](../5.5.1-SNS-2-Channels-By-Severity/)), Alarms ([5.5.2](../5.5.2-CloudWatch-Alarms/)) và Dashboard ([5.5.3](../5.5.3-Dashboard-Custom-Metrics/)), bước cuối là xác nhận chuỗi cảnh báo thực sự chạy — phần hay bị bỏ qua vì hạ tầng "trông ổn" ngay cả khi subscription email còn Pending hoặc Chatbot chưa gắn Slack.

#### Kịch bản test

| #   | Kịch bản                                                                      | Kỳ vọng                                                                                                                   |
| --- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| 1   | SNS Console → subscription `alerts-info`                                      | Trạng thái `Confirmed` (không phải `Pending confirmation`)                                                                |
| 2   | AWS Chatbot → Slack channel configuration                                     | Đúng workspace/channel; nếu `slack_*` = `NONE` thì resource Chatbot không tồn tại (đúng thiết kế)                         |
| 3   | Giả lập lỗi Lambda (throw exception liên tục vài phút, vượt ngưỡng Errors)    | `lambda-errors` → `ALARM`, nhận email qua `alerts-info`                                                                   |
| 4   | Gây lỗi 5xx phía API/Lambda (ví dụ transient fault trong chat-engine trả 500) | `apigw-5xx` → `ALARM` khi tỷ lệ 5xx > ngưỡng %, nhận email                                                                |
| 5   | Gây `ThrottlingException` (spam request Bedrock hoặc mock lỗi trong code)     | Log filter bắt chuỗi, `bedrock-throttle` → `ALARM`, tin nhắn trên Slack (`alerts-critical`)                               |
| 6   | Đẩy 1 message lỗi vào DLQ (ingestion DLQ hoặc function DLQ)                   | `dlq-depth` → `ALARM` ngay khi depth > 0, tin nhắn Slack                                                                  |
| 7   | Mở dashboard `${name_prefix}-overview`                                        | 7 widget AWS có dữ liệu sau khi có traffic; Cache Hit Rate sau vài lần gọi `/chat`; RAGAS sau khi evaluation chạy ≥ 1 lần |
| 8   | Dừng giả lập lỗi sau khi đã `ALARM`                                           | Alarm tự về `OK` khi hết vi phạm ngưỡng — không cần reset thủ công                                                        |

{{% notice tip %}}
Kịch bản 4: request sai path/method thường ra **4xx**, không kích hoạt `apigw-5xx`. Cần lỗi phía server (5xx) hoặc tỷ lệ 5xx đủ lớn trong cửa sổ 5 phút.
{{% /notice %}}

#### Kết quả đạt được

- Hai kênh SNS theo severity — Warning (email) và Critical (Slack) — giảm alarm fatigue.
- `bedrock-throttle` tận dụng log Lambda (Luồng 2) thay vì hạ tầng theo dõi throttle riêng.
- `dlq-depth` gộp cả 2 tầng DLQ (Luồng 1) — không bỏ sót message ở một tầng.
- Dashboard 9 widget: 7 metric AWS + cache hit rate (EMF) + RAGAS (`put_metric_data`).
- `treat_missing_data = "notBreaching"` tránh cảnh báo giả khi môi trường ít traffic.
