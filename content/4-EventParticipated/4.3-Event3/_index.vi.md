---
title: "Event 3"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 4.3. </b> "
---

# Bài thu hoạch: “Agent Forge - Deepdive Day 2”

### 1. Tổng quan về sự kiện
- **Thời gian**: 9:00AM - 12:00PM, Thứ 7, 08/08/2026
- **Địa điểm**: Tầng 26, Bitexco Financial Tower, 2 Đ. Hải Triều, Sài Gòn, Hồ Chí Minh 700000, Việt Nam.
- **Vai trò**: Người tham dự

### 2. Danh sách diễn giả
- **Nghia Tran** - Agentic SA
- **Anh Pham** - Cloud Consultant G-AsiaPacific Vietnam

---

### 3. Nội dung chính

#### Phần lý thuyết
Phần lý thuyết bao gồm các phần chính sau:

##### Memory
Memory giúp Agent lưu giữ thông tin, vượt qua giới hạn context window và cá nhân hóa trải nghiệm.
- **Short-term Memory**: Lưu dữ liệu thô từ hội thoại, đồng bộ để truy xuất nhanh thông tin gần nhất.
- **Long-term Memory**: Trích xuất insight và tri thức từ hội thoại, chuyển thành vector để lưu trữ lâu dài.
- **Memory Strategies**: Gồm Summary, User Preference, Semantic và Episodic.
- **Namespace**: Tổ chức dữ liệu theo cấu trúc phân cấp như `/Strategy/Actor/Session`, giúp thu hẹp phạm vi tìm kiếm, giảm token và tăng tốc truy xuất.

##### Evaluations
Evaluations đảm bảo Agent hoạt động chính xác, hữu ích và an toàn, đồng thời phát hiện hallucination, lỗi reasoning và lựa chọn tool không phù hợp.
- **Có hai chế độ**:
  - **On-demand Evaluation**: Đánh giá chủ động trong quá trình development.
  - **Online Evaluation**: Giám sát liên tục trong production thông qua telemetry và metrics.
- **Đánh giá được thực hiện ở ba cấp**:
  - **Session level**: Đánh giá toàn bộ phiên.
  - **Trace level**: Đánh giá từng response.
  - **Span level**: Đánh giá việc sử dụng tool và parameters.
- **Cơ chế**: Hệ thống sử dụng Judge để phân tích hoạt động của Agent, sau đó đưa kết quả vào Observability để SME theo dõi và can thiệp.

##### Observability
Observability giúp developer hiểu, debug và tối ưu hoạt động bên trong của Agent.
- **Ba thành phần chính**:
  - **Logs**: Cho biết điều gì đã xảy ra.
  - **Traces**: Cho biết quá trình xảy ra như thế nào.
  - **Metrics**: Đo lường tác động như latency, token cost và error rate.
- **Tính năng mở rộng**: Ngoài ra có OpenTelemetry, monitoring thời gian thực, alert và cơ chế phân cấp dữ liệu theo `Session` → `Trace` → `Span/Sub-span`.

##### AgentCore Components
Các component chính gồm:
- **Registry**: Trung tâm quản lý và tái sử dụng Agent skills, tools và APIs; hỗ trợ Admin, Publisher và Consumer.
- **Harness**: Framework tối giản để khởi tạo Agent từ Model + System Prompt + Tool, đồng thời hỗ trợ khả năng mở rộng.
- **Tools**: Giúp Agent tương tác với hệ thống bên ngoài, thực hiện actions và truy cập dữ liệu/API thời gian thực.
- **Payments**: Cho phép Agent thực hiện thanh toán, hiện hỗ trợ Stripe và Coinbase.
- **Optimization**: Sử dụng dữ liệu từ Evaluation và Observability để tìm điểm cần cải thiện, hỗ trợ A/B testing, Red Teaming và self-optimizing loop.
- **Policy**: Lớp kiểm soát hành vi, bảo mật và compliance của Agent; hỗ trợ Human-in-the-loop, Cedar, Strict/Permissive mode và nguyên tắc Least Privilege.

#### Phần thực hành
Hướng dẫn kỹ thuật triển khai với Agent SDK, thiết lập AWS Bedrock, và cách sử dụng công cụ dòng lệnh (CLI) để tạo project, deploy và test Agent trên AWS.

---

### 4. Bài học rút ra
Qua sự kiện **Agent Forge - Deepdive Day 2**, em hiểu rõ hơn về các thành phần cần thiết để xây dựng và vận hành một AI Agent trong môi trường production, đặc biệt là vai trò của Memory, Evaluations và Observability trong việc duy trì ngữ cảnh, đánh giá chất lượng và giám sát hoạt động của Agent. Không chỉ thế, em cũng hiểu được cách các thành phần của AgentCore như Registry, Harness, Tools, Policy và Optimization phối hợp với nhau để quản lý, mở rộng, bảo mật và liên tục cải thiện Agent. Đặc biệt, em nhận thức được tầm quan trọng của Least Privilege và Human-in-the-loop trong việc kiểm soát các hành động của Agent. Cuối cùng, phần thực hành giúp em làm quen với Agent SDK, AWS Bedrock và AWS CLI, đồng thời hiểu được quy trình cơ bản từ khởi tạo project, triển khai đến kiểm thử Agent trên AWS.

#### Một số hình ảnh khi tham gia sự kiện

<div style="display: flex; gap: 12px; justify-content: space-between; align-items: center; margin: 15px 0;">
  <img src="/images/4-EventParticipated/event3_01.jpg" alt="Hình ảnh tham gia sự kiện 1" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event3_02.jpg" alt="Hình ảnh tham gia sự kiện 2" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
  <img src="/images/4-EventParticipated/event3_03.jpg" alt="Hình ảnh tham gia sự kiện 3" style="width: 32%; height: auto; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); object-fit: cover;" />
</div>
