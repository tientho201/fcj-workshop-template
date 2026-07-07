---
title: "Worklog Tuần 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu tuần 1:

Nắm vững nền tảng quản trị & bảo mật tài khoản AWS (IAM, chi phí, hỗ trợ), hiểu và triển khai được hệ thống mạng cơ bản (VPC), làm quen môi trường phát triển trên cloud (Cloud9) và lưu trữ web tĩnh + database quan hệ (S3, RDS).

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc                                                                                                                                                                    | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu                                                                                                  |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | --------------------------------------------------------------------------------------------------------------- |
| 2   | - Access Management với IAM: user, group, policy, MFA <br> - Managing Costs với AWS Budgets: tạo budget alert <br> - Getting Help với AWS Support: các tier support, mở case | 22/06/2026   | 22/06/2026      | <https://000002.awsstudygroup.com/> , <https://000007.awsstudygroup.com/> , <https://000009.awsstudygroup.com/> |
| 3   | - Networking Essentials với Amazon VPC <br> - Thực hành: tạo VPC, subnet (public/private), Internet Gateway, Route Table, Security Group, NACL                               | 23/06/2026   | 23/06/2026      | <https://000003.awsstudygroup.com/>                                                                             |
| 4   | - Instance Profiling với IAM Roles for EC2 <br> - Cloud Development với AWS Cloud9 <br> - Thực hành: gắn IAM Role vào EC2, code trực tiếp trên Cloud9                        | 24/06/2026   | 24/06/2026      | <https://000048.awsstudygroup.com/> , <https://000049.awsstudygroup.com/>                                       |
| 5   | - Static Website Hosting với Amazon S3 <br> - Thực hành: tạo bucket, bật static hosting, upload file, cấu hình bucket policy public                                          | 25/06/2026   | 25/06/2026      | <https://000057.awsstudygroup.com/>                                                                             |
| 6   | - Database Essentials với Amazon RDS <br> - Thực hành: tạo RDS instance (MySQL/PostgreSQL), kết nối từ EC2, backup/snapshot cơ bản                                           | 26/06/2026   | 26/06/2026      | <https://000005.awsstudygroup.com/>                                                                             |

### Kết quả đạt được tuần 1:

- Quản lý được người dùng/quyền hạn qua IAM (user, group, policy, role), hiểu nguyên tắc least privilege.

- Thiết lập được cảnh báo chi phí (Budgets) và biết cách sử dụng AWS Support khi gặp sự cố.

- Tự thiết kế và triển khai được một VPC hoàn chỉnh: subnet, routing, gateway, security.

- Gắn IAM Role vào EC2 để cấp quyền an toàn mà không cần access key; làm quen IDE cloud Cloud9.

- Host được một website tĩnh trên S3.

- Triển khai và kết nối thành công database RDS từ ứng dụng/EC2.
