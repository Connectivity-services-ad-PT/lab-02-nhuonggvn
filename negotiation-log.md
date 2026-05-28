# Biên bản đàm phán hợp đồng API

- **Cặp đàm phán**: Pair 02 — Core Business (A6/B6) ↔ AI Vision (A4/B4)
- **Product**: Product A
- **Provider**: AI Vision (A4) - Đại diện: Nguyễn Minh Mạnh
- **Consumer**: Core Business (A6) - Đại diện: Nguyễn Thị Hồng Duyên
- **Phiên**: v1.0
- **Ngày**: 28-05-2026

---

## Issue #1: Định dạng dữ liệu hình ảnh (imageUrl vs imageBase64)

- **Raised by**: Provider (AI Vision)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Provider lo ngại rằng nếu gửi ảnh dạng `imageBase64` trực tiếp trong request body, kích thước request sẽ rất lớn (lên tới 10MB+ cho mỗi frame chất lượng cao), gây tải nặng cho network bandwidth và buffer bộ nhớ của server AI Vision. Provider đề xuất chỉ hỗ trợ `imageUrl` (ảnh được upload lên CDN/Object Storage dùng chung).
- **Proposal**: Consumer giải thích rằng trong một số tình huống khẩn cấp hoặc khi camera mất kết nối Internet ngoại vi, camera chỉ có thể chụp frame local và đẩy trực tiếp chuỗi Base64 lên Core Business. Do đó, Consumer đề xuất hỗ trợ cả hai: `imageUrl` (ưu tiên hàng đầu) và `imageBase64` (phương án dự phòng).
- **Resolution**: Accepted with Modification
- **Rationale**: Hai bên thống nhất sử dụng cấu trúc `oneOf` kết hợp `discriminator` với thuộc tính phân loại `imageType` (`URL` hoặc `BASE64`) trong schema `VisionDetectRequest`. Đồng thời, thiết lập giới hạn cứng kích thước chuỗi base64 tối đa là 10MB để tránh tấn công từ chối dịch vụ (DoS/Out of memory).
- **Impact**:
  - **AI Vision**: Cấu hình middleware giới hạn tối đa request body là 10MB.
  - **Core Business**: Cấu hình luồng nghiệp vụ ưu tiên upload CDN lấy `imageUrl`, chỉ sử dụng `imageBase64` khi CDN gặp sự cố.

---

## Issue #2: Cơ chế xử lý Đồng bộ (Synchronous) vs Bất đồng bộ (Asynchronous)

- **Raised by**: Consumer (Core Business)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Consumer yêu cầu tốc độ phản hồi nhanh (đồng bộ) để kịp thời ra quyết định đóng/mở cổng hoặc gửi cảnh báo an ninh tức thời. Tuy nhiên, Provider giải thích rằng việc phân tích Deep Learning (như chạy YOLOv8x) có thể mất từ 200ms đến hơn 2s tùy thuộc vào tải GPU hiện tại. Nếu bắt buộc xử lý đồng bộ khi tải cao sẽ gây ra nghẽn hàng đợi request và timeout.
- **Proposal**: Consumer đề xuất thiết kế API hỗ trợ đồng thời cả hai mã trạng thái phản hồi:
  1. Trả về `201 Created` kèm kết quả nhận dạng tức thời (luồng Đồng bộ khi GPU rảnh).
  2. Trả về `202 Accepted` kèm header `Location` trỏ đến đường dẫn tra cứu kết quả (luồng Bất đồng bộ khi GPU tải cao).
- **Resolution**: Accepted
- **Rationale**: Thiết kế lai này giúp hệ thống co giãn tốt, tránh nghẽn luồng mạng khi lượng request đột biến, đồng thời đảm bảo Core Business vẫn có thể nhận kết quả nhanh nhất khi hệ thống bình thường.
- **Impact**:
  - **AI Vision**: Thiết kế thêm cơ chế background queue xử lý ngầm và sinh thêm endpoint tra cứu `GET /vision/detect/{requestId}`.
  - **Core Business**: Thiết kế Rule Engine hỗ trợ nhận kết quả tức thời (201) hoặc chuyển sang luồng Polling định kỳ (202) để lấy kết quả từ endpoint GET.

---

## Issue #3: Bổ sung tọa độ đối tượng nhận dạng (Bounding Box)

- **Raised by**: Consumer (Core Business)
- **Endpoint**: `POST /vision/detect` & `GET /vision/detect/{requestId}`
- **Concern**: Thiết kế API ban đầu của AI Vision chỉ trả về danh sách nhãn nhận diện (`type`) và độ tin cậy (`confidence`). Tuy nhiên, Consumer cần vẽ lại khung nhận dạng trên Dashboard của Admin để làm bằng chứng an ninh trực quan (đặc biệt là vùng xâm nhập trái phép hoặc khuôn mặt đối tượng lạ).
- **Proposal**: Consumer yêu cầu Provider bổ sung thêm thông tin tọa độ và kích thước khung giới hạn (`boundingBox`) cho mỗi đối tượng nhận dạng được.
- **Resolution**: Accepted
- **Rationale**: Việc hiển thị trực quan Bounding Box là tính năng cực kỳ quan trọng đối với Dashboard giám sát, giúp tăng trải nghiệm người dùng và độ chính xác khi kiểm chứng an ninh.
- **Impact**:
  - **AI Vision**: Trích xuất thêm thông tin tọa độ `x`, `y`, `width`, `height` từ output của model AI và định nghĩa schema `BoundingBox` trong response.
  - **Core Business**: Cập nhật Processing DB để lưu trữ thông tin tọa độ này.

---

## Issue #4: Chuẩn hóa định dạng phản hồi lỗi (Problem Details - RFC 7807)

- **Raised by**: Consumer (Core Business)
- **Endpoint**: Tất cả endpoints
- **Concern**: Consumer muốn có một cấu trúc phản hồi lỗi thống nhất và giàu ngữ cảnh để dễ dàng xử lý tự động trong code và hiển thị chi tiết lỗi cho quản trị viên, thay vì mỗi lỗi trả về một kiểu (như plain text hoặc JSON thiếu cấu trúc).
- **Proposal**: Consumer đề xuất áp dụng chuẩn **Problem Details (RFC 7807)** với định dạng Content-Type là `application/problem+json` cho toàn bộ mã lỗi `4xx` và `5xx`.
- **Resolution**: Accepted
- **Rationale**: Đảm bảo tính chuyên nghiệp và đồng bộ trên toàn bộ kiến trúc Smart Campus Platform.
- **Impact**:
  - **Cả hai bên**: Thống nhất cấu trúc schema `Problem` bao gồm các trường: `type` (URI lỗi), `title` (tiêu đề lỗi), `status` (HTTP code), `detail` (mô tả lỗi chi tiết), `instance` (URI xảy ra lỗi) và mảng `errors` (danh sách lỗi chi tiết ở từng field cụ thể).

---

## Issue #5: Chống trùng lặp yêu cầu xử lý (Idempotency)

- **Raised by**: Provider (AI Vision)
- **Endpoint**: `POST /vision/detect`
- **Concern**: Khi kết nối mạng chập chờn hoặc xảy ra hiện tượng timeout giả, Core Business có thể tự động gửi lại (retry) cùng một yêu cầu phân tích. Nếu không chống trùng lặp, AI Vision sẽ phải chạy model phân tích lại từ đầu trên cùng một frame ảnh, gây lãng phí tài nguyên GPU cực kỳ lớn.
- **Proposal**: Provider yêu cầu Consumer bắt buộc phải tạo và đính kèm một ID duy nhất (`requestId` định dạng UUID) cho mỗi frame ảnh gửi lên. AI Vision sẽ dựa vào ID này để cache và kiểm tra trùng lặp.
- **Resolution**: Accepted
- **Rationale**: Bảo vệ tài nguyên GPU của hệ thống tránh quá tải vô ích bởi các request retry lặp lại từ phía Consumer.
- **Impact**:
  - **Core Business**: Đảm bảo sinh chuỗi UUID ngẫu nhiên duy nhất cho mỗi yêu cầu phân tích mới và truyền vào trường `requestId`.
  - **AI Vision**: Cấu hình cache kiểm tra `requestId`. Nếu request trùng lặp đang được xử lý, trả về mã lỗi `409 Conflict`. Nếu đã xử lý xong từ trước, trả về trực tiếp kết quả đã lưu trong database mà không cần chạy lại model AI.

---

## Issue #6: Đồng bộ hóa nhãn nhận dạng (Labels) và phiên bản Model AI

- **Raised by**: Consumer (Core Business)
- **Endpoint**: `GET /vision/models`
- **Concern**: AI Vision liên tục nâng cấp các phiên bản model nhận dạng (ví dụ từ YOLOv8 lên YOLOv9) dẫn đến việc thay đổi hoặc bổ sung thêm các nhãn nhận diện mới (như thêm nhãn `WEAPON` hoặc `FIRE`). Nếu Core Business không biết trước điều này, Rule Engine có thể xử lý sai hoặc bỏ qua các cảnh báo nguy hiểm do chưa được cấu hình nhãn tương ứng.
- **Proposal**: Consumer đề xuất AI Vision cung cấp một endpoint trả về danh sách các model AI đang chạy kèm theo phiên bản của chúng. Đồng thời, trong kết quả phân tích ảnh bắt buộc phải đính kèm trường thông tin `modelVersion`.
- **Resolution**: Accepted
- **Rationale**: Giúp Core Business tự động kiểm soát tính tương thích và cập nhật các cấu hình quy tắc (Rule Engine) một cách chủ động mà không cần can thiệp thủ công vào code khi AI Vision cập nhật model.
- **Impact**:
  - **AI Vision**: Triển khai thêm endpoint `/vision/models` trả về thông tin chi tiết của các model đang kích hoạt và bổ sung trường `modelVersion` vào schema `VisionDetectResult`.
  - **Core Business**: Sử dụng thông tin phiên bản model để ghi log kiểm toán (audit log) và đưa ra quyết định cảnh báo phù hợp.

---

# Chốt hợp đồng v1.0

- **Provider sign-off**: Nguyễn Minh Mạnh (Đại diện nhóm A4 AI Vision)
- **Consumer sign-off**: Nguyễn Thị Hồng Duyên (Đại diện nhóm A6 Core Business)
- **Witness (GV/TA)**: Lê Thái Bảo
- **Date**: 28-05-2026

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có warning nào | Hợp đồng API đã sạch lỗi và cảnh báo hoàn toàn sau khi kiểm thử. | Không cần sửa đổi bổ sung. |
