---
title: "Kiểm thử thủ công E2E"
date: 2024-01-01
weight: 1
chapter: false
pre: " <b> 5.10.1 </b> "
---

{{% notice note %}}
📌 Đây là phần **thật sự đã xảy ra** trong quá trình xây dựng, **không phải suy đoán** — nhưng chạy bằng script tạm trong scratchpad, **không commit vào repo**, nên không lặp lại tự động được. Khác hẳn kiểm tra tĩnh trong CI ([5.9.1](../../5.9-CICD/5.9.1-CI-Workflow/)) hay unit test tự động ([5.8.4](../../5.8-Backend/5.8.4-Backend-Testing/)) — cả 2 đều chạy lại giống nhau mỗi lần — nội dung trang này chỉ là **bằng chứng tại một thời điểm**, không phải quy trình có thể tái sử dụng.
{{% /notice %}}

#### Dựng tay 3 loại PDF test — phủ đúng 3 nhánh logic

Để test đúng cả 3 đường rẽ nhánh trích xuất văn bản của `_process_s3_object` (đã mô tả ở [Luồng 1, trang 5.3.3](../../5.3-Data-Ingestion/5.3.3-Text-Extraction/)) — chứ không chỉ test "chạy không lỗi" — cần dựng tay 3 loại PDF khác nhau:

| Loại PDF                             | Cách dựng                                                                                     | Nhánh logic test được                                              |
| ------------------------------------ | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **PDF có lớp text nhúng sẵn**        | Viết tay từng object/xref/trailer PDF **ở mức byte** — vì máy không có sẵn `reportlab`/`fpdf` | `pypdf` đọc được text trực tiếp, không cần OCR                     |
| **PDF chỉ chứa ảnh, không có text**  | `PIL.Image.save(..., "PDF")`                                                                  | `pypdf` trả về chuỗi rỗng → chuyển `awaiting_ocr_confirmation`     |
| **PDF ảnh có chữ vẽ thật trong ảnh** | `PIL.ImageDraw`/`ImageFont` vẽ chữ lên ảnh trước khi lưu thành PDF                            | Textract có **nội dung thật** để OCR ra, không chỉ ảnh trắng/nhiễu |

```python
# Ví dụ dựng PDF loại 3 (ảnh có chữ thật) - minh họa cách tiếp cận
from PIL import Image, ImageDraw, ImageFont

img = Image.new("RGB", (800, 1000), color="white")
draw = ImageDraw.Draw(img)
draw.text((50, 50), "Đây là nội dung test OCR thật", fill="black")
img.save("test-scanned-with-real-text.pdf", "PDF")
```

{{% notice tip %}}
Việc **viết tay từng object/xref/trailer ở mức byte** cho PDF loại 1 là chi tiết đáng chú ý nhất — cho thấy mức độ hiểu sâu định dạng file PDF (không chỉ dùng thư viện có sẵn), cần thiết vì môi trường phát triển không có `reportlab`/`fpdf` cài sẵn. Đây là điểm kỹ thuật đáng đưa vào báo cáo như minh chứng cho khả năng giải quyết vấn đề khi công cụ tiêu chuẩn không có sẵn.
{{% /notice %}}

#### Script gọi API thật, không mock

Một script Python (dùng `requests`) **tự viết lại đúng luồng `cognitoLogin()`** mà UI dùng (`InitiateAuth` với `USER_PASSWORD_AUTH` — xem lại [Frontend, trang 5.7.1](../../5.7-Frontend/5.7.1-Frontend-Architecture-Authentication/)), dùng để test `/documents`, `/status`, `/documents-decision`, `/chat` **end-to-end trên hạ tầng AWS thật**, không mock bất kỳ phần nào.

```python
# Minh họa cấu trúc script test thật (không phải test suite chính thức)
def cognito_login(username, password):
    response = requests.post(
        f"https://cognito-idp.{region}.amazonaws.com/",
        headers={"X-Amz-Target": "AWSCognitoIdentityProviderService.InitiateAuth", ...},
        json={"AuthFlow": "USER_PASSWORD_AUTH", ...},
    )
    return response.json()["AuthenticationResult"]["IdToken"]

token = cognito_login(test_user, test_password)
upload_result = requests.post(f"{api_url}/documents", headers={"Authorization": token}, json={...})
# ... tiếp tục test /status, /documents-decision, /chat
```

#### Bug thật được test bắt ra — không phải giả định

{{% notice warning %}}
📌 **Đây là bằng chứng cụ thể nhất cho giá trị của kiểm thử thủ công:** gọi `/documents-decision` với `decision: "ocr"` lần đầu **trả về `504`** — không phải lỗi giả định. Truy vết lỗi này dẫn thẳng tới phát hiện **thiếu VPC endpoint cho Lambda control-plane API** (đã kể chi tiết ở [Luồng 2, trang 5.4.6](../../5.4-Realtime-QA/5.4.6-Error-Handling-OCR-Decision/) và [Backend, trang 5.8.4](../../5.8-Backend/5.8.4-Backend-Testing/)).
{{% /notice %}}

Sau khi thêm `aws_vpc_endpoint "lambda"` và `apply` lại, **test lại xác nhận đúng luồng resume OCR chạy hết toàn bộ**:

```
confirm → Textract OCR → chunk → embed → index → completed
```

Đây chính là ví dụ rõ ràng nhất trong toàn dự án cho thấy **kiểm thử thủ công, dù không tự động, vẫn có giá trị phát hiện lỗi thật** — nếu không có bước test end-to-end trên hạ tầng thật này, bug thiếu VPC endpoint có thể đã không được phát hiện cho tới khi người dùng thật gặp phải.

---
