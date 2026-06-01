# Event Contract sơ bộ — Pair 05: IoT Ingestion ↔ Core Business

> **Lưu ý**: Bản hợp đồng sự kiện (Event Contract) này dùng để ghi nhận các thỏa thuận kỹ thuật ban đầu giữa nhóm **IoT Ingestion (A1)** và **Core Business (A6)** cho luồng tích hợp bất đồng bộ qua Queue (Queue async). Đặc tả chi tiết bằng AsyncAPI sẽ được thực hiện chi tiết trong Lab 03.

---

## 1. Thông tin Dependency

- **Cặp tích hợp**: Pair 05 — IoT Ingestion (A1/B1) ↔ Core Business (A6/B6)
- **Publisher (Producer)**: IoT Ingestion (A1)
- **Subscriber (Consumer)**: Core Business (A6)
- **Cơ chế truyền thông**: Queue async (Message Broker / Kafka hoặc RabbitMQ)
- **Ngày thống nhất**: 01-06-2026
- **Đại diện hai bên**:
  - Đại diện IoT Ingestion (A1): 
  - Đại diện Core Business (A6): Nguyễn Thị Hồng Duyên

---

## 2. Mục đích nghiệp vụ

IoT Ingestion thu thập dữ liệu thô từ các cảm biến vật lý (nhiệt độ, độ ẩm, khói, trạng thái cửa...) và publish các sự kiện tương ứng vào hàng đợi. **Core Business (A6)** tiêu thụ (consume) các sự kiện này để:
1. Ghi nhận telemetry lịch sử của campus.
2. Đánh giá tính hợp lệ của các chính sách vận hành (Policy Engine).
3. Phát hiện lỗi thiết bị hoặc các tình huống bất thường (như cháy nổ, xâm nhập trái phép) để đưa ra hành động phản hồi realtime (như đóng cửa, gửi cảnh báo).

---

## 3. Danh sách Event & Topic dự kiến

| Event Name | Topic / Routing Key | Mô tả nghiệp vụ | Tần suất |
| :--- | :--- | :--- | :--- |
| `sensor.reading.created` | `iot.campus.sensor.reading` | Gửi định kỳ khi cảm biến ghi nhận dữ liệu telemetry mới. | Định kỳ (mỗi 10s - 60s/thiết bị) |
| `sensor.threshold.exceeded` | `iot.campus.sensor.alert` | Kích hoạt ngay lập tức khi một chỉ số vượt quá ngưỡng an toàn được cấu hình. | Bất thường (realtime khi có sự cố) |

---

## 4. Payload chi tiết và Cấu trúc dữ liệu mẫu

Hai bên thống nhất sử dụng chuẩn **CloudEvents v1.0** để bọc gói tin (metadata) kết hợp với phần nghiệp vụ cụ thể nằm trong thuộc tính `data`.

### 4.1. Event: `sensor.reading.created`
Được kích hoạt khi cảm biến gửi chỉ số định kỳ.

```json
{
  "eventId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "eventType": "sensor.reading.created",
  "occurredAt": "2026-06-01T10:45:00.123Z",
  "correlationId": "a85cfc8a-7b3b-419b-a068-15cf464971c2",
  "source": "/campus/iot-ingestion/gateways/gw-01",
  "data": {
    "deviceId": "temp-sensor-h302",
    "sensorType": "TEMPERATURE",
    "value": 28.5,
    "unit": "CELSIUS",
    "locationId": "room-h302-floor-3",
    "status": "NORMAL"
  }
}
```

### 4.2. Event: `sensor.threshold.exceeded`
Được kích hoạt tức thì khi cảm biến phát hiện vượt ngưỡng an toàn (ví dụ: nhiệt độ quá cao, phát hiện khói, rò rỉ khí).

```json
{
  "eventId": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
  "eventType": "sensor.threshold.exceeded",
  "occurredAt": "2026-06-01T10:46:12.987Z",
  "correlationId": "e2bbcb44-77db-4e26-bb2b-7cf0fbf3a44d",
  "source": "/campus/iot-ingestion/gateways/gw-01",
  "data": {
    "deviceId": "smoke-sensor-hallway-2",
    "sensorType": "SMOKE",
    "value": 85.0,
    "unit": "PERCENTAGE",
    "locationId": "hallway-floor-2-building-a",
    "thresholdLimit": 50.0,
    "severity": "CRITICAL"
  }
}
```

---

## 5. Ràng buộc kỹ thuật & Nghiệp vụ thống nhất

| Vấn đề đàm phán | Quyết định thống nhất | Rationale (Lý do kỹ thuật) |
| :--- | :--- | :--- |
| **Hệ thống đơn vị (Unit)** | Sử dụng hệ SI quốc tế: `CELSIUS` cho nhiệt độ, `PERCENTAGE` cho độ ẩm/khói, `PASCAL` cho áp suất, `LUX` cho ánh sáng. | Đảm bảo tính nhất quán trong xử lý logic của Policy Engine mà không cần parse đơn vị thủ công. |
| **Định danh vị trí (`locationId`)** | **Bắt buộc**. Phải có định danh cụ thể của vị trí gắn thiết bị trong trường `data.locationId`. | Giúp Core Business kích hoạt đúng chính sách của phòng học/tòa nhà đó (ví dụ: kích hoạt báo cháy cho khu vực cụ thể). |
| **Cơ chế chống lặp (Idempotency)** | Core Business (Consumer) sẽ cache các `eventId` nhận được trong vòng **24 giờ** để loại bỏ các event bị xử lý lặp do retry của broker. | Phòng ngừa tình trạng Broker gửi trùng lặp thông điệp (At-Least-Once Delivery). |
| **Độ trễ tối đa (TTL)** | Nếu độ trễ nhận tin ($t_{\text{receive}} - t_{\text{occurredAt}}$) **vượt quá 5 phút**, Core Business sẽ bỏ qua xử lý nghiệp vụ chính và chỉ ghi nhận log. | Tránh việc hệ thống kích hoạt các policy khẩn cấp quá hạn khi mạng bị nghẽn cục bộ rồi thông suốt trở lại. |
| **Đảm bảo thứ tự (Ordering)** | IoT Ingestion phải ghi nhận chính xác thứ tự thông điệp theo thiết bị bằng cách sử dụng `deviceId` làm **Partition Key** (nếu dùng Kafka) hoặc Message Group ID. | Giúp Core Business luôn cập nhật trạng thái mới nhất của thiết bị, tránh tình trạng nhận tin cũ sau tin mới. |

---

## 6. Các vấn đề chuyển tiếp sang Lab 03 (Đặc tả chi tiết với AsyncAPI)

1. **Schema Validation nâng cao**: Khai báo chi tiết định dạng JSON Schema cho từng thuộc tính cụ thể trong `data` (sử dụng AsyncAPI 3.0 Components).
2. **Cấu hình Retry Policy và DLQ (Dead Letter Queue)**: Thống nhất số lần retry tối đa khi Consumer gặp lỗi nghiệp vụ (ví dụ: 3 lần trước khi đẩy vào DLQ `iot.campus.sensor.dlq`).
3. **Cơ chế Heartbeat / Keepalive**: Thỏa thuận về event kiểm tra trạng thái hoạt động trực tuyến của các Gateway IoT (`sensor.heartbeat.created`).
