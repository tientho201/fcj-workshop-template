---
title: "Cấu hình AWS Credentials"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.2.1 </b> "
---


### Các bước chuẩn bị

**Bước 1.1 - Đăng nhập Console và chọn Region**
 
Đăng nhập [AWS Management Console](https://console.aws.amazon.com/), chọn Region **us-east-1 (N. Virginia)** ở góc trên phải.

![Chọn Region ở góc trên phải AWS Console](/images/5-Workshop/5.2-Prerequisite/image5.2.1-1.png)

**Bước 1.2 - Bật MFA cho root**

Vào **IAM → Dashboard → Security recommendations → Add MFA**, dùng app xác thực (Google Authenticator/Authy) quét QR code.

![Cấu hình MFA cho root user](/images/5-Workshop/5.2-Prerequisite/image5.2.1-2.png)


**Bước 1.3 - Thêm Policies bằng file JSON**

Copy các thông tin trong **IAM permissions**. Sau đó vào **IAM → Dashboard**. Sau đó **Menu bên trái → Policies → bấm nút Create policy (góc trên phải)**. Bạn sẽ thấy 2 tab: **Visual editor** và **JSON** - bấm chọn tab **JSON**. Xóa hết đoạn code mẫu có sẵn trong ô đó, dán **(Ctrl+V)** toàn bộ nội dung bạn vừa copy ở bước 1 vào. Bấm **Next**. Ở màn hình tiếp theo, có 1 ô Policy name — đây là nơi duy nhất bạn cần tự đặt tên, ví dụ gõ: ``RAGProjectDeployPolicy``.
Bấm **Create policy**.

![Tạo policies](/images/5-Workshop/5.2-Prerequisite/image5.2.1-3a.png)
![Tạo policies](/images/5-Workshop/5.2-Prerequisite/image5.2.1-3b.png)
![Tạo policies](/images/5-Workshop/5.2-Prerequisite/image5.2.1-3c.png)

→ Xong bước này, policy đã được tạo với đầy đủ cả 8 nhóm quyền bên trong (bạn không cần thêm gì nữa).

**Bước 1.4 - Gắn policy vào Group**
 
Vào **IAM → IAM User groups → Create group** (ví dụ `rag-developers`). **Tab Permissions → Add permissions → Attach policies**. Gõ tìm đúng tên bạn đặt ở bước 7 **(RAGProjectDeployPolicy) → tick chọn → Add permissions**.

![Tạo IAM Group và gắn policy](/images/5-Workshop/5.2-Prerequisite/image5.2.1-4.png)

**Bước 1.5 - Tạo IAM User và thêm vào group**

Vào trong **IAM → IAM Users → Create user**, thêm vào group, bật Console access + bắt buộc MFA.

![Tạo IAM User và thêm vào group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5a.png)

![Tạo IAM User và thêm vào group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5b.png)

![Tạo IAM User và thêm vào group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5c.png)

![Tạo IAM User và thêm vào group](/images/5-Workshop/5.2-Prerequisite/image5.2.1-5d.png)

**Bước 1.6 — Tạo Access Key và cấu hình AWS CLI**
 
Tạo Access key cho IAM User **(Security credentials → Create access key)**

![Cấu hình AWS  với access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6a.png)

![Cấu hình AWS  với access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6b.png)

![Cấu hình AWS với access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6c.png)

![Cấu hình AWS với access key](/images/5-Workshop/5.2-Prerequisite/image5.2.1-6d.png)

**Bước 1.7 — Cấu hình AWS Budgets**
 
Vào **Billing and Cost Management → Budgets → Create budget**, loại **Cost budget**, ngưỡng theo tháng, cảnh báo ở 50%/80%/100%, nhập email cả nhóm.
 
![Tạo Budget cảnh báo chi phí](/images/5-Workshop/5.2-Prerequisite/image5.2.1-7a.png)

![Tạo Budget cảnh báo chi phí](/images/5-Workshop/5.2-Prerequisite/image5.2.1-7b.png)

![Tạo Budget cảnh báo chi phí](/images/5-Workshop/5.2-Prerequisite/image5.2.1-7c.png)


#### Nội dung tiếp theo

- [Cấu hình HCP Terraform](../5.2.2-Configure-HCP-Terraform/_index.vi.md)

