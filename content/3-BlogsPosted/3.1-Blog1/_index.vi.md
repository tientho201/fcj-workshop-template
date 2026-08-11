---
title: "Blog 1"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# Theo AI Engineer thì nên ưu tiên học gì trong AWS Study Group?

Xin chào mọi người,

Khi tham gia AWS Study Group, mình thấy lộ trình học khá đầy đủ, từ những kiến thức nền tảng như IAM, VPC, EC2, S3, cho đến các dịch vụ AI/ML như Amazon SageMaker.

Ban đầu mình cũng có suy nghĩ là phải học hết tất cả theo đúng thứ tự. Tuy nhiên, vì mục tiêu của mình là trở thành AI Engineer nên mình tự đặt ra một câu hỏi:
_Nếu thời gian học có hạn thì đâu là những nội dung mình nên ưu tiên trước để phục vụ công việc AI?_

Sau khi xem toàn bộ roadmap và tìm hiểu thêm về các dịch vụ của AWS, đây là những gì mình rút ra.

## Đầu tiên, nền tảng AWS vẫn rất quan trọng

Mình không nghĩ AI Engineer chỉ cần biết Machine Learning là đủ.

Trong thực tế, một mô hình AI muốn hoạt động thì vẫn cần có nơi lưu trữ dữ liệu, nơi triển khai mô hình, hệ thống phân quyền và nhiều dịch vụ hỗ trợ khác.
Vì vậy, các dịch vụ như:

- Amazon S3
- IAM
- EC2

vẫn là những kiến thức mình sẽ học đầy đủ.
Tuy nhiên, nếu phải sắp xếp mức độ ưu tiên thì mình sẽ dành nhiều thời gian hơn cho các dịch vụ AI/ML.

## Amazon SageMaker – Dịch vụ mình ưu tiên nhất

Có lẽ đây là phần mình mong chờ nhất trong AWS Study Group.
Trước đây khi làm các project Machine Learning hay Deep Learning, quy trình của mình thường là:
**Dataset → Tiền xử lý dữ liệu → Train Model → Đánh giá → Lưu model**
Hầu hết đều thực hiện trên máy cá nhân bằng PyTorch hoặc Scikit-learn.

Khi tìm hiểu về SageMaker, mình nhận ra AWS đã tích hợp gần như toàn bộ quy trình này trên một nền tảng duy nhất.
Trong workshop Immersion Day, mình sẽ được thực hành:

- Chuẩn bị dữ liệu
- Feature Engineering
- Phân tích dữ liệu
- Train mô hình XGBoost
- Tự động tối ưu Hyperparameter
- Deploy model thành Endpoint

Điều này giúp mình hiểu rõ hơn cách một mô hình Machine Learning được xây dựng và triển khai trong môi trường thực tế.

## Feature Engineering – Không chỉ là xử lý dữ liệu

Trước đây mình thường sử dụng Pandas hoặc Scikit-learn để làm các bước tiền xử lý dữ liệu.

Trong SageMaker, AWS cung cấp Data Wrangler để hỗ trợ trực quan hóa dữ liệu, phân tích tương quan giữa các đặc trưng và xây dựng quy trình Feature Engineering trước khi huấn luyện mô hình.

Điều mình thấy thú vị là dữ liệu sau khi xử lý có thể lưu vào Feature Store để tái sử dụng cho nhiều mô hình khác nhau.

Đây là khái niệm mình chưa có cơ hội sử dụng khi làm các project cá nhân nên rất muốn tìm hiểu thêm.

## Automatic Hyperparameter Tuning

Khi làm Deep Learning, chắc hẳn ai cũng từng phải thử rất nhiều giá trị khác nhau cho:

- Learning Rate
- Batch Size
- Optimizer
- Epoch

Mỗi lần thay đổi lại phải train lại mô hình.

Trong SageMaker, AWS hỗ trợ Automatic Model Tuning giúp tự động tìm kiếm bộ Hyperparameter phù hợp thay vì phải thử thủ công từng cấu hình.

Theo mình, đây là một tính năng rất hữu ích khi làm việc với các bài toán Machine Learning trong thực tế.

## Deploy Model – Bước mà nhiều sinh viên thường bỏ qua

Mình nhận thấy khi học Machine Learning ở trường, phần lớn bài tập thường kết thúc sau khi mô hình đạt Accuracy tốt.

Tuy nhiên, trong doanh nghiệp thì đó mới chỉ là một phần của quy trình.
Sau khi huấn luyện xong, mô hình còn cần được triển khai để các ứng dụng khác có thể gửi dữ liệu và nhận kết quả dự đoán.

Thông qua SageMaker Endpoint, mình có cơ hội hiểu rõ hơn cách một mô hình được đưa vào sử dụng thay vì chỉ chạy trên máy cá nhân.

Theo mình, đây là một kỹ năng rất quan trọng đối với AI Engineer.

## Những nội dung mình vẫn sẽ học

Điều đó không có nghĩa là mình sẽ bỏ qua những module khác.
Các kiến thức như:

- Networking
- VPC
- IAM
- EC2
- Monitoring
  đều là nền tảng để xây dựng các hệ thống AI trên AWS.

Chỉ là ở giai đoạn hiện tại, mình sẽ ưu tiên học sâu hơn những nội dung liên quan trực tiếp đến Machine Learning trước, sau đó sẽ quay lại bổ sung các phần còn lại để có cái nhìn đầy đủ hơn.

## Cuối cùng

Sau khi xem roadmap của AWS Study Group, mình nhận ra AI Engineer ngày nay không chỉ cần biết xây dựng mô hình mà còn cần hiểu cách dữ liệu được quản lý, cách mô hình được triển khai và cách các dịch vụ AI vận hành trên Cloud.

Đó cũng là lý do mình lựa chọn ưu tiên các nội dung liên quan đến Amazon SageMaker, Feature Engineering, Hyperparameter Tuning và Model Deployment.

Hy vọng sau khi hoàn thành các workshop này, mình sẽ không chỉ biết train một mô hình AI mà còn hiểu được cách đưa mô hình đó vào phục vụ trong các ứng dụng thực tế.

Nếu mọi người cũng đang theo đuổi AI Engineer hoặc Machine Learning Engineer, rất mong được trao đổi thêm về cách học AWS sao cho phù hợp với mục tiêu của mỗi người.
Cảm ơn mọi người đã đọc bài viết!

## Link

<https://www.facebook.com/groups/awsstudygroupfcj/permalink/2224802604951366/?rdid=chYR8yCqrR14Gdm0#>
