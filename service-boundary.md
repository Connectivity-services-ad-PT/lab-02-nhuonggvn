# Service Boundary của nhóm

## 1. Thông tin nhóm

- Tên nhóm: A6
- Lớp: CNTT17-11
- Thành viên: Trần Công Thưởng,Nguyễn Văn Hưởng,Nguyễn Thị Hồng Duyên,Nguyễn Công Khánh
- Service nhóm phụ trách: Core business (A6)
- Sản phẩm tổng thể của lớp: ProductA

## 2. Actor

Ai tương tác với hệ thống/service?

- **User/Admin**: Người dùng hoặc quản trị viên tương tác trực tiếp với Core Business để thực hiện các thao tác cấu hình, quản lý và xác thực.

## 3. System Boundary

Nhóm em xây phần nào?

**Phần nhóm kiểm soát:**
- **A6 Core Business**: Bao gồm toàn bộ các logic cốt lõi của hệ thống:
  - Business Logic (Logic nghiệp vụ)
  - Rule Engine (Công cụ xử lý luật)
  - Workflow Manager (Quản lý luồng công việc)
  - Auth/Permission (Xác thực và Phân quyền)
- **Processing DB**: Cơ sở dữ liệu lưu trữ và xử lý thông tin cho Core Business (sử dụng PostgreSQL hoặc MySQL).

**Phần nhóm chỉ tích hợp (giao tiếp với A6):**
- **A1 IoT Ingestion**: Tiếp nhận dữ liệu từ các thiết bị IoT và đẩy vào Core Business.
- **A4 AI Vision Image Analysis**: Dịch vụ phân tích hình ảnh bằng AI (tích hợp qua REST sync).
- **A3 Access Gate**: Hệ thống đóng/mở cổng tự động (A6 gọi REST sync để điều khiển).
- **A5 Analytics Reports/Dash**: Hệ thống báo cáo và hiển thị dữ liệu phân tích (A6 đồng bộ dữ liệu sang).
- **A7 Notification**: Dịch vụ gửi thông báo qua Email, SMS hoặc ứng dụng di động (A6 gửi sự kiện qua Queue async).

**Phần nhóm KHÔNG tích hợp vào A6 (chỉ để tham khảo kiến trúc tổng thể):**
- **A2 Camera Stream**: Luồng truyền video/hình ảnh từ camera (giao tiếp trực tiếp với A4 và A5, không liên quan A6).

## 4. Service Boundary

**Service của nhóm có trách nhiệm gì?**
- Tiếp nhận dữ liệu đã qua xử lý từ A1 (IoT Ingestion).
- Tiếp nhận kết quả phân tích hình ảnh từ A4 (AI Vision) đẩy về qua REST sync.
- Xử lý các logic nghiệp vụ (Business Logic) và áp dụng các bộ quy tắc (Rule Engine).
- Điều phối luồng công việc (Workflow Manager) giữa các dịch vụ.
- Quản lý định danh, xác thực và phân quyền truy cập (Auth/Permission) cho User/Admin.
- Lưu trữ và truy vấn dữ liệu từ cơ sở dữ liệu chính (Processing DB).
- Ra lệnh đóng/mở cổng đến A3 (Access Gate) qua REST sync.
- Kích hoạt gửi thông báo đến A7 (Notification) qua Queue async.
- Đồng bộ dữ liệu nghiệp vụ phục vụ hiển thị/báo cáo sang A5 (Analytics).

**Service KHÔNG làm gì?**
- Không trực tiếp thu thập dữ liệu thô từ phần cứng IoT (trách nhiệm của A1).
- Không xử lý luồng camera hay phân tích nhận diện khuôn mặt/biển số (trách nhiệm của A2 và A4).
- Không trực tiếp điều khiển đóng/mở mạch điện của cổng (trách nhiệm của A3).
- Không đảm nhận việc render biểu đồ hoặc tổng hợp báo cáo chuyên sâu (trách nhiệm của A5).
- Không trực tiếp cấu hình hạ tầng gửi SMS/Email (trách nhiệm của A7).
- **Không giao tiếp với A2 Camera Stream** (A2 chỉ giao với A4 và A5).

## 5. Input / Output

### Input
- Dữ liệu IoT từ dịch vụ **A1 IoT Ingestion**.
- Kết quả phân tích hình ảnh/khuôn mặt/biển số từ dịch vụ **A4 AI Vision Image Analysis** (qua REST sync).
- Yêu cầu cấu hình, quản trị, đăng nhập từ **User/Admin**.
- Dữ liệu trạng thái phản hồi từ **Processing DB**.

### Output
- Lệnh điều khiển đóng/mở cổng gửi tới **A3 Access Gate** (qua REST sync).
- Sự kiện (Event) gửi thông báo đẩy vào hàng đợi của **A7 Notification** (qua Queue async).
- Dữ liệu đồng bộ phục vụ báo cáo gửi tới **A5 Analytics Reports/Dash**.
- Phản hồi kết quả xử lý, cấp quyền cho **User/Admin**.

## 6. API dự kiến

| Method | Endpoint | Mục đích | Ghi chú |
|--------|----------|----------|---------|
| GET | /health | Kiểm tra trạng thái hoạt động của Core Business service | Do A6 cung cấp |
| POST | /api/v1/auth/login | Xác thực người dùng và cấp quyền (Auth/Permission) | Do A6 cung cấp |
| POST | /api/v1/vision-callback | Tiếp nhận kết quả phân tích hình ảnh từ A4 AI Vision | Do A6 cung cấp |
| POST | /api/v1/iot/data | Tiếp nhận dữ liệu gửi về từ A1 IoT Ingestion | Do A6 cung cấp |
| GET | /api/v1/workflows | Quản lý và kiểm tra các luồng công việc (Workflow Manager) | Do A6 cung cấp |

**API mà A6 gọi sang service khác (không phải do A6 cung cấp):**
- `POST /api/v1/control/gate` (gọi sang A3 Access Gate qua REST sync)

## 7. Phụ thuộc service khác

**Service này gọi đến service nào?**
- Gọi đến **A3 Access Gate** (REST sync) để ra lệnh điều khiển đóng/mở cổng.
- Gọi đồng bộ dữ liệu sang **A5 Analytics Reports/Dash** (REST sync / API nội bộ).
- Đẩy message sự kiện sang **A7 Notification** (Queue async) để thực hiện gửi thông báo.
- Đọc/ghi dữ liệu liên tục vào **Processing DB (PostgreSQL/MySQL)**.

**Service nào gọi đến service này?**
- **User/Admin** tương tác trực tiếp qua giao diện UI/Client.
- **A1 IoT Ingestion** gọi API để truyền dữ liệu cảm biến/thiết bị vào hệ thống.
- **A4 AI Vision Image Analysis** gọi webhook/callback để trả kết quả phân tích hình ảnh.

**Service khác trong hệ thống (không liên quan trực tiếp đến A6, chỉ để tham khảo):**
- A2 Camera Stream → gửi dữ liệu đến A4 và A5.
- A1 IoT Ingestion → cũng gửi dữ liệu qua queue async tới A5 (không qua A6).
- A3 Access Gate → cũng gửi trạng thái qua queue async tới A5 (không qua A6).

## 8. Sơ đồ minh họa

```mermaid
flowchart TD
    User[User/Admin] <-->|Tương tác/Auth| A6[A6 Core Business]
    
    A1[A1 IoT Ingestion] -->|Dữ liệu IoT| A6
    A1 -.->|Queue async| A5
    
    A2[A2 Camera Stream] --> A4
    A2 --> A5
    
    A4[A4 AI Vision] <-->|REST sync| A6
    
    A6 <--> DB[(Processing DB)]
    A6 -->|REST sync| A3[A3 Access Gate]
    A6 -->|Queue async| A7[A7 Notification]
    A6 -->|Đồng bộ dữ liệu| A5[A5 Analytics Reports/Dash]
    
    A3 -.->|Queue async| A5
    
    A5 --> User