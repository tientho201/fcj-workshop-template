---
title: "Event 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.2. </b> "
---

# Bài thu hoạch: “Agent Forge - Deepdive Day 1”

### 1. Tổng quan về sự kiện
- **Thời gian**: 9:00AM - 12:00PM, Thứ 7, 01/08/2026
- **Địa điểm**: Tầng 26, Bitexco Financial Tower, 2 Đ. Hải Triều, Sài Gòn, Hồ Chí Minh 700000, Việt Nam.
- **Vai trò**: Người tham dự

### 2. Danh sách diễn giả
- **Nghia Tran** - Agentic SA
- **Anh Pham** - Cloud Consultant G-AsiaPacific Vietnam

---

### 3. Nội dung chính
Đây là buổi workshop chuyên sâu (L300) về Amazon Bedrock Agent Core, dành cho các doanh nghiệp muốn xây dựng hệ thống AI tự chủ (Agentic AI) ở quy mô sản xuất (production-ready).

#### Phần lý thuyết
Phần lý thuyết bao gồm các phần chính sau:

- **Giới thiệu về Agentic AI**: Giải thích khái niệm AI tự chủ, khả năng lập kế hoạch và thực thi công việc theo từng bước, cũng như mức độ tự chủ từ *deterministic workflow* (quy trình định sẵn) đến *fully autonomous*.
- **Amazon Bedrock Agent Core**: Dịch vụ giúp quản lý, triển khai và vận hành các tác nhân AI (agents). Hệ thống này tuân thủ các tiêu chuẩn công nghiệp về hiệu năng, khả năng mở rộng (scalability) và bảo mật.
- **Các thành phần cốt lõi của Agent Core**:
  - **Runtime Environment**: Môi trường serverless để chạy agent, sử dụng công nghệ Firecracker MicroVM để cách ly tài nguyên.
  - **Identity**: Lớp quản lý xác thực (authentication) và phân quyền (authorization) bằng JSON Web Tokens (JWT) và Workload Access Token.
  - **Gateway**: Lớp trung gian (middleware) để kết nối agent với các công cụ (tools) khác nhau thông qua giao thức MCP (Model Context Protocol), hỗ trợ kiểm soát tập trung và tính năng human-in-the-loop.
- **Bảo mật và Kết nối**: Thảo luận về việc kết nối agent với mạng VPC, sử dụng AWS PrivateLink để đảm bảo dữ liệu nội bộ không bị lộ ra ngoài mạng công cộng.

#### Phần thực hành
Tập trung vào việc hướng dẫn mọi người thực hành các dịch vụ Runtime, Gateway, Identity đã được đề cập về lý thuyết ở trước cũng như chỉ người tham dự cách cài đặt các môi trường cần thiết cho phần thực hành.

---

### 4. Bài học rút ra
Qua sự kiện **Agent Forge - Deepdive Day 1** này, em hiểu rõ hơn về khái niệm Agentic AI và cách xây dựng các AI Agent có khả năng tự lập kế hoạch, thực thi nhiệm vụ và tương tác với các dịch vụ bên ngoài. Em cũng nắm được kiến trúc của Amazon Bedrock Agent Core, bao gồm các thành phần Runtime, Identity và Gateway, cùng vai trò của từng thành phần trong quá trình triển khai và vận hành AI Agent. Bên cạnh đó, em nhận thức được tầm quan trọng của bảo mật khi triển khai hệ thống AI thông qua việc kết hợp Amazon VPC và AWS PrivateLink. Phần thực hành giúp em làm quen với quy trình cấu hình môi trường và triển khai các dịch vụ của Bedrock Agent Core, từ đó hiểu rõ hơn mối liên hệ giữa lý thuyết và ứng dụng thực tế.

#### Một số hình ảnh khi tham gia sự kiện

<div style="display: flex; gap: 12px; justify-content: space-between; align-items: center; margin: 15px 0;">
  <img src="/images/4-EventParticipated/event2_01.jpg" alt="Hình ảnh tham gia sự kiện 1" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event2_02.jpg" alt="Hình ảnh tham gia sự kiện 2" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event2_03.jpg" alt="Hình ảnh tham gia sự kiện 3" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
</div>
