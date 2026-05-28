# Chiến lược Quản lý Phiên bản API (API Versioning Strategy)

Tài liệu này đặc tả chiến lược quản lý phiên bản (Versioning) cho API giữa **Core Business (A6)** và **AI Vision (A4)** (Pair 02) nhằm đảm bảo hệ thống vận hành liên tục và giảm thiểu tối đa rủi ro gián đoạn khi cập nhật hoặc nâng cấp dịch vụ.

---

## 1. Nguyên tắc Định danh Phiên bản (Semantic Versioning)

Chúng tôi tuân thủ triệt để nguyên tắc **Semantic Versioning (SemVer)** cho URL và HTTP Headers:
* **Cú pháp**: `v{MAJOR}` (Ví dụ: `/api/v1/` trong thực tế hoặc quản lý hợp đồng thông qua `version: 1.0.0` trong OpenAPI).
* **MAJOR (Phiên bản lớn)**: Tăng khi có các thay đổi **không tương thích ngược (Breaking Changes)**. Yêu cầu thay đổi URL hoặc nâng cấp lớn ở phía Consumer.
* **MINOR (Phiên bản phụ)**: Tăng khi thêm các tính năng mới hoặc thuộc tính mới **tương thích ngược (Backward-Compatible Changes)**.
* **PATCH (Vá lỗi)**: Tăng khi sửa lỗi nội bộ của API mà không làm thay đổi cấu trúc dữ liệu gửi nhận.

---

## 2. Phân loại Thay đổi (Change Classification)

### 2.1. Thay đổi Tương thích Ngược (Backward-Compatible Changes)
* **Thêm Endpoint mới**: Ví dụ thêm `GET /vision/models` để kiểm tra model đang chạy mà không ảnh hưởng đến các request phân tích ảnh đang có.
* **Thêm thuộc tính tùy chọn (Optional Fields) vào Request**: Thêm trường `correlationId` hoặc `metadata` trong `VisionDetectRequest`.
* **Thêm thuộc tính vào Response**: Thêm trường `processingTimeMs` hoặc `notes` trong `VisionDetectResult`.
* **Bổ sung nhãn nhận diện mới vào Enum**: Thêm nhãn nhận diện `FIRE` hoặc `SMOKE` vào danh sách kết quả (Consumer cần viết Rule Engine thông minh bỏ qua các nhãn lạ chưa cấu hình thay vì bị crash ứng dụng).

### 2.2. Thay đổi Không Tương thích Ngược (Breaking Changes)
* **Xóa hoặc đổi tên Endpoint**: Đổi tên `/vision/face-match` thành `/vision/detect`.
* **Xóa hoặc đổi tên trường dữ liệu bắt buộc**: Đổi tên trường `requestId` thành `id`.
* **Thay đổi kiểu dữ liệu bắt buộc**: Thay đổi `capturedAt` từ định dạng chuỗi `date-time` sang số nguyên `timestamp`.
* **Thay đổi cấu trúc phản hồi lỗi**: Thay đổi từ định dạng lỗi phẳng sang định dạng lỗi chuẩn hóa **Problem Details (RFC 7807)**.
* **Thay đổi HTTP Authentication**: Chuyển từ Bearer Token sang OAuth2.

---

## 3. Quy trình Ngừng hoạt động API (API Deprecation & Sunset)

Khi một API hoặc thuộc tính bị thay thế bởi giải pháp tốt hơn, chúng tôi áp dụng quy trình **Deprecation** và **Sunset** chuyên nghiệp thay vì đột ngột đóng kết nối.

### 3.1. Đánh dấu Deprecated trong OpenAPI
* Sử dụng thuộc tính `deprecated: true` tại mức Path Operation hoặc Property Schema trong file `openapi.yaml`.
* **Ví dụ thực tế trong hợp đồng v1.0**:
  * **Legacy Path**: `/vision/face-match` được đánh dấu `deprecated: true` do được gộp và thay thế bởi API `/vision/detect` đa hình và tối ưu hơn.
  * **Schema Property**: Trường `status` trong `ModelInfo` được đánh dấu `deprecated: true` để chuyển sang dùng các API quản lý trạng thái hạ tầng chi tiết hơn.

### 3.2. Cảnh báo qua HTTP Headers (Sunset & Deprecation)
Khi Consumer gọi vào các API đã bị deprecated, hệ thống Provider sẽ trả về các Header cảnh báo trong response:
1. **`Deprecation`**: Chỉ thị rằng API này đã bị deprecated.
   ```http
   Deprecation: Thu, 28 May 2026 15:00:00 GMT
   ```
2. **`Sunset`**: Xác định thời điểm chính thức ngừng hoạt động và xóa bỏ hoàn toàn API này khỏi hệ thống.
   ```http
   Sunset: Thu, 31 Dec 2026 23:59:59 GMT
   ```
   *(Đã được khai báo trong phần header response mẫu của `/vision/face-match` trong `openapi.yaml`)*.
3. **`Link`**: Cung cấp liên kết dẫn tới tài liệu hướng dẫn chuyển dịch sang API mới.
   ```http
   Link: <https://api.campus.local/docs/migration-v2>; rel="successor-version"
   ```

---

## 4. Lộ trình chuyển dịch v1.0 -> v2.0 của Pair 02

1. **Giai đoạn 1 (Hiện tại - Đàm phán và chạy song song)**:
   * Giữ lại `/vision/face-match` (đánh dấu `deprecated: true`).
   * Khuyến khích Core Business tích hợp hoàn toàn qua `/vision/detect` (hỗ trợ cả URL và Base64).
2. **Giai đoạn 2 (Thông báo hoàng hôn - Sunset Warning)**:
   * Trả về header `Sunset: Thu, 31 Dec 2026 23:59:59 GMT` cho mọi request gọi vào `/vision/face-match`.
   * Gửi cảnh báo tự động về hệ thống log của Admin Core Business để thúc giục chuyển dịch.
3. **Giai đoạn 3 (Chấm dứt hoạt động - EOL)**:
   * Vào ngày **01-01-2027**, chính thức ngắt kết nối endpoint `/vision/face-match` (trả về lỗi `410 Gone` hoặc `404 Not Found`).
