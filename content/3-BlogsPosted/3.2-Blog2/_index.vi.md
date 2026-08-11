---
title: "Blog 2"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.2. </b> "
---

# Theo Data Engineer thì nên ưu tiên học gì trong AWS Study Group?

Xin chào mọi người,

Khi tham gia AWS Study Group, mình thấy lộ trình học khá đầy đủ, từ những kiến thức nền tảng như IAM, VPC, EC2, S3, cho đến các dịch vụ nâng cao về Data và AI/ML.

Ban đầu mình cũng có suy nghĩ là phải học hết tất cả theo đúng thứ tự. Tuy nhiên, vì mục tiêu của mình là trở thành Data Engineer nên mình tự đặt ra một câu hỏi:
_"Nếu thời gian học có hạn thì đâu là những nội dung mình nên ưu tiên trước để phục vụ công việc Data Engineering khi đi làm?"_

Sau khi xem toàn bộ roadmap và tìm hiểu thêm về các dịch vụ của AWS, đây là những gì mình rút ra.

## Đầu tiên, nền tảng AWS vẫn là bước khởi đầu bắt buộc

Một hệ thống dữ liệu (Data Pipeline) dù hiện đại đến đâu thì nền tảng bên dưới vẫn phải dựa trên bộ khung lưu trữ, tính toán và bảo mật của AWS.
Vì vậy, các dịch vụ như:

- Amazon S3: Nơi đóng vai trò làm Data Lake chính cho mọi luồng dữ liệu.
- AWS IAM: Quản lý truy cập, phân quyền an toàn cho từng Service và User.
- Amazon EC2 & VPC: Hiểu cấu trúc mạng và máy chủ để vận hành các tool/agent tự dựng nếu cần.
  vẫn là những kiến thức mình bắt buộc phải học vững. Tuy nhiên, nếu phải sắp xếp mức độ ưu tiên để ứng dụng cho công việc Data Engineer thực tế, mình sẽ dành nhiều thời gian hơn cho bộ dịch vụ xử lý, lưu trữ và luồng dữ liệu (Data Engineering Ecosystem).

## Amazon S3 & Data Lake Architecture – Dịch vụ ưu tiên hàng đầu

Trước đây khi làm các bài tập hay project cá nhân, dữ liệu thường nằm trong các file CSV/JSON trên máy hoặc một cơ sở dữ liệu quan hệ nhỏ.
Tuy nhiên, trong thực tế enterprise, Amazon S3 đóng vai trò vô cùng cốt lõi:

- Tổ chức kiến trúc Data Lake theo các tầng dữ liệu (Bronze/Raw, Silver/Cleaned, Gold/Curated).
- Quản lý định dạng file tối ưu cho truy vấn đại dữ liệu như Parquet, ORC, Avro.
- Tối ưu chi phí lưu trữ với các tầng Lifecycle Rules (S3 Standard, S3 Glacier).

Hiểu và áp dụng đúng S3 như một Data Lake là kỹ năng đầu tiên một Data Engineer cần nắm chắc.

## AWS Glue & ETL Pipelines – Trái tim của xử lý dữ liệu

Nếu như ở máy cá nhân chúng ta quen viết script Python đơn lẻ để làm sạch dữ liệu, thì ở quy mô doanh nghiệp, AWS Glue chính là công cụ đắc lực:

- AWS Glue Data Catalog & Crawler: Tự động phát hiện schema và quản lý metadata cho toàn bộ Data Lake.
- Serverless Spark (Glue ETL Jobs): Xử lý và biến đổi các tập dữ liệu khổng lồ (Batch Processing) mà không cần tự dựng cluster Spark thủ công.
- Glue Workflow / Orchestration: Kết nối các bước tiền xử lý, kiểm tra chất lượng dữ liệu và nạp dữ liệu thành một pipeline tự động.

Nắm vững AWS Glue giúp Data Engineer giải quyết bài toán biến đổi dữ liệu một cách linh hoạt và mở rộng (scalable).

## Amazon Redshift – Data Warehouse cho phân tích & Báo cáo

Sau khi dữ liệu đã được làm sạch trên Data Lake, công việc tiếp theo là nạp vào Data Warehouse để phục vụ cho BI Dashboard hay các truy vấn kinh doanh phức tạp.

Amazon Redshift là dịch vụ mình cực kỳ quan tâm:

- Nắm vững kiến trúc lưu trữ dạng cột (Columnar Storage) và phân tán dữ liệu (Distribution Keys, Sort Keys) để tối ưu tốc độ truy vấn SQL.
- Tích hợp Redshift Spectrum giúp truy vấn trực tiếp dữ liệu trên S3 mà không cần load hoàn toàn vào kho dữ liệu.
- Thiết kế data model (Star Schema, Snowflake Schema) chuẩn hóa cho các báo cáo doanh nghiệp.

## Amazon Kinesis / Amazon MSK – Xử lý dữ liệu thời gian thực (Real-time Streaming)

Trong nhiều bài toán thực tế (định danh sự kiện người dùng, theo dõi log hệ thống, giao dịch tài chính), dữ liệu không chỉ đến theo từng đợt (Batch) mà chảy liên tục theo dòng (Stream).

Việc tiếp cận các dịch vụ như Amazon Kinesis (Data Streams, Firehose) hoặc Amazon MSK (Managed Streaming for Apache Kafka) giúp Data Engineer:

- Xây dựng kiến trúc xử lý dữ liệu thời gian thực (Real-time Ingestion).
- Đưa dữ liệu luồng trực tiếp về Data Lake (S3) hoặc kho dữ liệu tức thì.

## Amazon Athena & AWS Lake Formation – Phân tích nhanh & Bảo mật dữ liệu

- Amazon Athena: Cho phép chạy truy vấn SQL ad-hoc trực tiếp trên file S3 mà không cần dựng database. Đây là công cụ cực kỳ tiện lợi để khám phá (explore) dữ liệu thô.
- AWS Lake Formation: Giúp thiết lập chính sách bảo mật, phân quyền chi tiết tới cấp dòng/cột (Row/Column-level security) trên Data Lake một cách đơn giản hơn việc cấu hình IAM thủ công.

## Những nội dung mình vẫn sẽ học bổ sung

Tất nhiên, làm Data Engineer không chỉ dừng lại ở các dịch vụ Data thuần túy. Những kiến thức như:

- **Networking & VPC Peering:** Để kết nối an toàn giữa Data Warehouse với cơ sở dữ liệu nội bộ (On-premises DB).
- **Amazon CloudWatch & AWS EventBridge:** Để giám sát pipeline, tự động cảnh báo khi job ETL gặp lỗi.
- **Infrastructure as Code (Terraform / AWS CDK):** Để quản lý toàn bộ hạ tầng Data Pipeline dưới dạng mã nguồn.

đều rất cần thiết. Nhưng ở giai đoạn hiện tại, ưu tiên hàng đầu của mình vẫn là làm chủ luồng đi của dữ liệu từ **Ingestion → Storage → Transformation → Warehouse**.

## Cuối cùng

Tham gia AWS Study Group giúp mình nhận ra Data Engineer không chỉ là viết query SQL hay code Python, mà là biết chọn đúng dịch vụ Cloud phù hợp cho từng bài toán dữ liệu, tối ưu được cả về hiệu năng (Performance) lẫn chi phí (Cost) cho doanh nghiệp.

Ưu tiên tập trung vào bộ dịch vụ S3, Glue, Redshift, Athena, Kinesis chính là con đường ngắn nhất giúp mình sẵn sàng cho các bài toán thực tế của một Data Engineer.

Nếu mọi người cũng đang định hướng theo Data Engineering hoặc Analytics Engineering, rất mong nhận được thêm chia sẻ và góc nhìn từ mọi người!
Cảm ơn mọi người đã đọc bài viết!

## Link

<https://www.facebook.com/groups/awsstudygroupfcj/permalink/2236445443787082/?rdid=XxPEuBNX5l2ZEBPZ#>
