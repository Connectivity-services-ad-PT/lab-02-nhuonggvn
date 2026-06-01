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

---

# PHẦN BỔ SUNG: Biên bản đàm phán hợp đồng Sự kiện (Event Contract)
## Cặp đàm phán: Pair 05 — IoT Ingestion (A1/B1) ↔ Core Business (A6/B6)

- **Cơ chế**: Queue async
- **Publisher (Producer)**: IoT Ingestion (A1)
- **Subscriber (Consumer)**: Core Business (A6)
- **Phiên**: v1.0 (Sơ bộ cho Lab 02)
- **Ngày**: 01-06-2026

---

## Issue #1: Tiêu chuẩn hóa hệ thống đơn vị đo lường (Metric Units)

- **Raised by**: Subscriber (Core Business)
- **Endpoint/Event**: `sensor.reading.created` & `sensor.threshold.exceeded`
- **Concern**: IoT Ingestion thu thập dữ liệu từ các cảm biến phần cứng của nhiều hãng sản xuất khác nhau, dẫn tới dữ liệu nhiệt độ có thể là độ Celsius (°C) hoặc Fahrenheit (°F), dữ liệu độ ẩm có thể là tỷ lệ phần trăm (0-100) hoặc số thực (0.0-1.0). Nếu gửi lung tung, Core Business sẽ không thể đánh giá chính xác các chính sách vận hành khẩn cấp trong Policy Engine.
- **Proposal**: Subscriber đề xuất bắt buộc chuẩn hóa toàn bộ giá trị đo lường (`value`) về hệ đo lường quốc tế (SI) cụ thể: `CELSIUS` cho nhiệt độ, `PERCENTAGE` (0.0-100.0) cho độ ẩm/khói, `PASCAL` cho áp suất, và `LUX` cho ánh sáng. Định dạng này phải được ghi nhận rõ trong trường `unit`.
- **Resolution**: Accepted
- **Rationale**: Đảm bảo tính nhất quán tuyệt đối của dữ liệu đầu vào, giúp bộ lọc quy tắc (Policy Engine) của Core Business tính toán nhanh chóng mà không mất tài nguyên convert đơn vị.
- **Impact**:
  - **IoT Ingestion**: Thực hiện chuyển đổi đơn vị ở tầng Gateway/Adapter trước khi đóng gói payload gửi lên hàng đợi.
  - **Core Business**: Định nghĩa Enum cho trường `unit` và xây dựng logic xử lý theo hệ đo lường chuẩn hóa.

---

## Issue #2: Yêu cầu định danh vị trí vật lý (`locationId`)

- **Raised by**: Subscriber (Core Business)
- **Endpoint/Event**: Cả hai sự kiện
- **Concern**: Thiết kế payload ban đầu của IoT Ingestion chỉ chứa `deviceId` và chỉ số đo được. Tuy nhiên, Core Business cần biết cảm biến đó đang được lắp đặt ở phòng học, hành lang hay tầng mấy để kích hoạt các kịch bản an ninh tương ứng (như mở/đóng cửa thoát hiểm của tòa nhà đó khi có cháy). Nếu chỉ có `deviceId`, Core Business sẽ phải truy vấn thêm DB để phân tích vị trí, gây trễ thời gian phản hồi.
- **Proposal**: Subscriber yêu cầu IoT Ingestion đính kèm thuộc tính `locationId` (kiểu chuỗi định danh vị trí) trực tiếp vào trong payload của mỗi sự kiện.
- **Resolution**: Accepted
- **Rationale**: Việc đính kèm `locationId` giúp Core Business xử lý logic tại chỗ (In-memory Rule Processing) cực kỳ nhanh chóng mà không cần thực hiện thêm các thao tác JOIN database đắt đỏ trong luồng realtime khẩn cấp.
- **Impact**:
  - **IoT Ingestion**: Bổ sung cơ chế làm giàu dữ liệu (Data Enrichment) tại tầng Ingestion để đính kèm `locationId` tương ứng với `deviceId` trước khi publish event.
  - **Core Business**: Cập nhật logic xử lý sự kiện để trích xuất trực tiếp `locationId` phục vụ Policy Engine.

---

## Issue #3: Giới hạn độ trễ xử lý thông điệp khẩn cấp (Event TTL)

- **Raised by**: Subscriber (Core Business)
- **Endpoint/Event**: Cả hai sự kiện
- **Concern**: Trong trường hợp Broker bị tắc nghẽn hoặc mất kết nối mạng cục bộ, các thông điệp có thể bị kẹt trong hàng đợi nhiều giờ. Khi mạng thông suốt, broker sẽ tự động đẩy hàng loạt thông điệp cũ lên Consumer. Nếu Core Business xử lý các sự kiện khẩn cấp cũ này (như báo khói cũ cách đây 2 tiếng), nó có thể kích hoạt các cảnh báo giả, hú còi và khóa cửa campus không đúng thời điểm hiện tại.
- **Proposal**: Subscriber đề xuất áp dụng luật TTL (Time-To-Live): Nếu thời gian nhận sự kiện trễ hơn quá **5 phút** so với thời điểm xảy ra sự kiện (`occurredAt`), Core Business sẽ ghi log cảnh báo và bỏ qua việc kích hoạt kịch bản khẩn cấp.
- **Resolution**: Accepted with Modification
- **Rationale**: Bảo vệ hệ thống Smart Campus khỏi các quyết định an ninh sai lệch do dữ liệu cũ (stale data), đồng thời vẫn lưu trữ thông tin phục vụ mục đích kiểm toán (audit log) và phân tích lịch sử sau này.
- **Impact**:
  - **Core Business**: Cấu hình logic lọc sự kiện: So sánh `timestamp_hiện_tại - occurredAt`. Nếu > 300 giây, bỏ qua xử lý Policy Engine, chỉ lưu DB kiểm toán.

---

## Ký kết đồng thuận Pair 05 (v1.0)

- **Publisher sign-off**: Nguyễn Tuấn Anh (Đại diện nhóm A1 IoT Ingestion)
- **Subscriber sign-off**: Nguyễn Thị Hồng Duyên (Đại diện nhóm A6 Core Business)
- **Witness (GV/TA)**: Lê Thái Bảo
- **Date**: 01-06-2026

---

# PHẦN BỔ SUNG: Biên bản đàm phán REST sync với Access Gate
## Cặp đàm phán: Pair 03 — Core Business (A6/B6) [Consumer] ↔ Access Gate (A3/B3) [Provider]

- **Cơ chế**: REST sync
- **Consumer**: Core Business (A6)
- **Provider**: Access Gate (A3)
- **Phiên**: v1.0
- **Ngày**: 01-06-2026

---

### Issue #1: Cơ chế phân trang danh sách nhật ký quẹt thẻ (`GET /access/logs/recent`)

- **Raised by**: Provider (Access Gate)
- **Concern**: Số lượng sự kiện quẹt thẻ phát sinh hàng ngày tại campus lên tới hàng chục nghìn lượt. Nếu trả về toàn bộ danh sách (Offset pagination thông thường), server Access Gate sẽ bị nghẽn bộ nhớ khi tính toán truy vấn `OFFSET` lớn, đồng thời gây trễ mạng nặng nề.
- **Proposal**: Provider đề xuất bắt buộc sử dụng cơ chế phân trang dựa trên con trỏ (Cursor-based Pagination). Consumer sẽ truyền tham số `cursor` (chuỗi base64 mã hóa mốc thời gian và ID) kết hợp tham số `limit` (giới hạn tối đa 100 bản ghi/trang).
- **Resolution**: Accepted
- **Rationale**: Phân trang bằng Cursor mang lại hiệu năng cao và ổn định ($O(1)$ thay vì $O(N)$), tránh tình trạng trùng lặp hoặc bỏ sót bản ghi khi có log mới chèn vào liên tục trong thời gian thực.
- **Impact**:
  - **Access Gate**: Cấu hình cơ sở dữ liệu và viết logic phân trang trả về trường `nextCursor` và `hasMore`.
  - **Core Business**: Điều chỉnh logic thu thập dữ liệu tự động (crawler/sync job) sử dụng vòng lặp dựa trên `nextCursor` thay vì chỉ số trang (page index).

---

### Issue #2: Quản lý trạng thái Thẻ RFID khi bị khóa hoặc hết hạn (`GET /cards/{cardId}`)

- **Raised by**: Consumer (Core Business)
- **Concern**: Khi một sinh viên bị kỷ luật hoặc thẻ bị mất, Admin sẽ thực hiện thao tác khóa thẻ. Core Business cần biết trạng thái chính xác của thẻ để hiển thị trên Dashboard và đồng bộ với Policy Engine.
- **Proposal**: Consumer yêu cầu thiết lập Enum cụ thể cho trường `status` của thẻ kiểm soát, bao gồm: `ACTIVE`, `BLOCKED`, `EXPIRED`. Tránh trả về kiểu Boolean đơn giản (như `isValid: false`) vì thiếu ngữ cảnh xử lý.
- **Resolution**: Accepted
- **Rationale**: Định nghĩa Enum rõ ràng giúp Core Business đưa ra cảnh báo chính xác cho nhân viên bảo vệ (ví dụ: thẻ này bị mất cắp nên bị Blocked, thay vì chỉ hiển thị là Thẻ không hợp lệ).
- **Impact**:
  - **Access Gate**: Cập nhật DB và Schema `CardDetail` định nghĩa rõ enum status.
  - **Core Business**: Cập nhật giao diện giám sát hiển thị màu sắc tương ứng theo trạng thái thẻ.

---

# PHẦN BỔ SUNG: Biên bản đàm phán REST sync kiểm tra Policy realtime
## Cặp đàm phán: Pair 10 — Access Gate (A3/B3) [Consumer] ↔ Core Business (A6/B6) [Provider]

- **Cơ chế**: REST sync
- **Consumer**: Access Gate (A3)
- **Provider**: Core Business (A6)
- **Phiên**: v1.0
- **Ngày**: 01-06-2026

---

### Issue #1: Ràng buộc về thời gian phản hồi (SLA Latency) của endpoint `/access/check`

- **Raised by**: Consumer (Access Gate)
- **Concern**: Khi sinh viên quẹt thẻ để đi qua cổng xoay (turnstile) hoặc barie, thời gian chờ tối đa từ lúc quẹt đến lúc cửa mở không được quá 1 giây để tránh ùn tắc cục bộ tại cổng trường vào giờ cao điểm. Do đó, thời gian xử lý API thẩm định quyền của Core Business bắt buộc phải cực kỳ tối ưu.
- **Proposal**: Consumer yêu cầu thiết lập giới hạn SLA cứng: thời gian phản hồi (latency) của API `/access/check` phải dưới **50ms** trong điều kiện bình thường và dưới **200ms** khi tải cao.
- **Resolution**: Accepted
- **Rationale**: Đảm bảo trải nghiệm người dùng mượt mà và tránh gây tắc nghẽn vật lý tại các khu vực cổng ra vào của campus.
- **Impact**:
  - **Core Business (Provider)**: Áp dụng cơ chế lưu trữ đệm trong bộ nhớ (In-memory caching/Redis) cho các chính sách hoạt động tích cực (`AccessPolicy`), thay vì thực hiện truy vấn SQL trực tiếp trên đĩa cứng cho mỗi lượt quẹt thẻ.
  - **Access Gate (Consumer)**: Cấu hình kết nối HTTP client timeout tối đa là 500ms.

---

### Issue #2: Cơ chế xử lý Sự cố khi kết nối bị lỗi hoặc Server sập (Fail-Safe Strategy)

- **Raised by**: Consumer (Access Gate)
- **Concern**: Nếu đường truyền mạng nội bộ giữa Access Gate và Core Business bị đứt, hoặc server Core Business gặp sự cố nghiêm trọng không thể phản hồi API `/access/check` (HTTP 5xx hoặc Timeout), các cổng vật lý sẽ xử lý thế nào? Nếu khóa cứng toàn bộ cổng (Fail-Closed) sẽ gây hỗn loạn và nguy hiểm nếu có sự cố cháy nổ. Nếu mở toang toàn bộ cổng (Fail-Open) sẽ gây rò rỉ an ninh nghiêm trọng.
- **Proposal**: Cả hai bên thảo luận và đề xuất giải pháp lai:
  1. Trạng thái mặc định khi lỗi là **Fail-Safe (Mở cưỡng bức)** đối với các cửa thoát hiểm (Emergency exits) hoặc trong khung giờ hành chính khi có bảo vệ trực.
  2. Áp dụng cơ chế lưu trữ offline cache tại Access Gate: Access Gate sẽ tự động lưu trữ danh sách thẻ VIP/Cán bộ được phép ra vào khẩn cấp để tự đưa ra quyết định offline khi không kết nối được tới Core Business.
- **Resolution**: Accepted
- **Rationale**: Giải pháp lai đảm bảo cân bằng tối ưu giữa tính an toàn tính mạng con người (Safety) và bảo mật tài sản campus (Security).
- **Impact**:
  - **Access Gate**: Triển khai module Offline Rule Engine cục bộ trên thiết bị điều khiển cổng để xử lý khi mất kết nối mạng.
  - **Core Business**: Định kỳ 1 tiếng/lần đồng bộ danh sách thẻ cán bộ và chính sách khẩn cấp xuống offline cache của các gateway cổng kiểm soát.

---

# PHẦN BỔ SUNG: Biên bản đàm phán Queue async khẩn cấp gửi Alert
## Cặp đàm phán: Pair 04 — Core Business (A6/B6) ↔ Notification (A7/B7)

- **Cơ chế**: Queue async
- **Publisher (Producer)**: Core Business (A6)
- **Subscriber (Consumer)**: Notification (A7)
- **Phiên**: v1.0
- **Ngày**: 01-06-2026

---

### Issue #1: Bảo vệ thông tin liên lạc nhạy cảm của người dùng (Data Privacy)

- **Raised by**: Subscriber (Notification)
- **Concern**: Theo luật an toàn thông tin và bảo mật dữ liệu học đường, việc truyền trực tiếp các thông tin nhạy cảm của sinh viên/giảng viên (như Email cá nhân, số điện thoại, token thiết bị di động) qua hàng đợi sự kiện `alert.triggered` là rất nguy hiểm vì hàng đợi có thể bị nghe lén hoặc ghi log thô (raw logging).
- **Proposal**: Subscriber đề xuất Core Business chỉ gửi mã định danh người dùng (`userId`, `role` hoặc `recipients.department`) trong payload sự kiện. Dịch vụ Notification sẽ tự động thực hiện truy vấn nội bộ DB bảo mật của mình để lấy thông tin liên lạc chi tiết trước khi gửi tin.
- **Resolution**: Accepted
- **Rationale**: Giảm thiểu tối đa nguy cơ rò rỉ thông tin cá nhân (PII), tách biệt rõ ràng trách nhiệm quản lý dữ liệu người dùng của từng microservice.
- **Impact**:
  - **Core Business**: Lọc bỏ các thông tin email, phone khỏi payload sự kiện `alert.triggered`.
  - **Notification**: Thiết lập module tra cứu liên lạc bảo mật tích hợp trực tiếp trong luồng xử lý gửi tin.

---

# PHẦN BỔ SUNG: Biên bản đàm phán Queue async Thống kê Analytics
## Cặp đàm phán: Pair 08 — Core Business (A6/B6) ↔ Analytics (A5/B5)

- **Cơ chế**: Queue async
- **Publisher (Producer)**: Core Business (A6)
- **Subscriber (Consumer)**: Analytics (A5)
- **Phiên**: v1.0
- **Ngày**: 01-06-2026

---

### Issue #1: Phân cấp độ ưu tiên xử lý thông điệp thống kê (Message Priority)

- **Raised by**: Publisher (Core Business)
- **Concern**: Dịch vụ Analytics tiêu thụ lượng lớn dữ liệu thống kê từ Core Business (mỗi lượt quẹt thẻ đều có 1 sự kiện `policy.evaluation.completed`). Khi có tải cao, hàng đợi thống kê này có thể tăng vọt số lượng thông điệp, gây tắc nghẽn Broker ảnh hưởng đến hàng đợi các tin nhắn khẩn cấp (như báo cháy `alert.triggered`).
- **Proposal**: Publisher đề xuất cấu hình tách biệt hoàn toàn hai Topic vật lý khác nhau trong hệ thống Message Broker:
  1. Topic khẩn cấp (`core.campus.alert.triggered`) chạy trên hàng đợi ưu tiên cao, được cấp băng thông GPU/CPU riêng biệt.
  2. Topic thống kê (`core.campus.policy.evaluation`) chạy trên hàng đợi ưu tiên thấp, cho phép có độ trễ tiêu thụ lên tới 10 phút.
- **Resolution**: Accepted
- **Rationale**: Đảm bảo cách ly lỗi và bảo vệ tính realtime của các chức năng an ninh tối quan trọng trước các hoạt động phân tích thống kê phi khẩn cấp.
- **Impact**:
  - **Cả hai bên**: Thiết lập kết nối tới hai topic/queue tách biệt trên Broker.

---

## Ký kết đồng thuận hợp nhất toàn diện Core Business (A6) v1.0

- **Đại diện Core Business (A6)**: Nguyễn Thị Hồng Duyên
- **Đại diện AI Vision (A4)**: Nguyễn Minh Mạnh (Pair 02)
- **Đại diện Access Gate (A3)**:  (Pair 03 & Pair 10)
- **Đại diện IoT Ingestion (A1)**: Nguyễn Tuấn Anh (Pair 05)
- **Đại diện Notification (A7)**: (Pair 04)
- **Đại diện Analytics (A5)**: (Pair 08)
- **Witness (GV/TA)**: Lê Thái Bảo
- **Ngày ký kết hợp nhất**: 01-06-2026
