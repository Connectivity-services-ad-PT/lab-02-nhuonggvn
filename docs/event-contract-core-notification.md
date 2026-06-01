# Event Contract sơ bộ — Pair 04: Core Business ↔ Notification

> **Lưu ý**: Bản hợp đồng sự kiện (Event Contract) này dùng để ghi nhận các thỏa thuận kỹ thuật ban đầu giữa nhóm **Core Business (A6)** và **Notification (A7)** cho luồng tích hợp bất đồng bộ qua Queue (Queue async). Đặc tả chi tiết bằng AsyncAPI sẽ được thực hiện chi tiết trong Lab 03.

---

## 1. Thông tin Dependency

- **Cặp tích hợp**: Pair 04 — Core Business (A6/B6) ↔ Notification (A7/B7)
- **Publisher (Producer)**: Core Business (A6)
- **Subscriber (Consumer)**: Notification (A7)
- **Cơ chế truyền thông**: Queue async (Message Broker / Kafka hoặc RabbitMQ)
- **Ngày thống nhất**: 01-06-2026
- **Đại diện hai bên**:
  - Đại diện Core Business (A6): Nguyễn Văn Hưởng
  - Đại diện Notification (A7): Trần Thị B (Ví dụ)

---

## 2. Mục đích nghiệp vụ

Khi bộ xử lý nghiệp vụ trung tâm (**Core Business - A6**) phát hiện một tình huống khẩn cấp hoặc sự kiện đặc biệt (như đột nhập, báo khói, quẹt thẻ sai nhiều lần, quyết định ra vào bất thường), nó sẽ publish một sự kiện thông báo tương ứng vào hàng đợi. **Notification (A7)** sẽ tiêu thụ (consume) sự kiện này để tự động định tuyến và gửi thông điệp cảnh báo tới người dùng cuối (Admin, Bảo vệ, Sinh viên) qua các kênh truyền thông phù hợp (SMS, Email, Mobile Push).

---

## 3. Danh sách Event & Topic dự kiến

| Event Name | Topic / Routing Key | Mô tả nghiệp vụ | Tần suất |
| :--- | :--- | :--- | :--- |
| `alert.triggered` | `core.campus.alert.triggered` | Kích hoạt ngay lập tức khi phát hiện tình huống khẩn cấp (cháy, đột nhập) cần gửi thông báo khẩn tới bảo vệ. | Bất thường (realtime khi có sự cố) |
| `notification.broadcast.requested` | `core.campus.notification.broadcast` | Gửi yêu cầu thông báo hàng loạt (ví dụ: đổi lịch học, thông báo đóng cửa campus). | Thấp (khi có sự kiện hành chính) |

---

## 4. Payload chi tiết và Cấu trúc dữ liệu mẫu

Hai bên thống nhất sử dụng chuẩn **CloudEvents v1.0** làm vỏ bọc ngoài.

### 4.1. Event: `alert.triggered`
Được publish bởi Core Business khi phát hiện sự cố an ninh nghiêm trọng từ AI Vision hoặc cảm biến IoT.

```json
{
  "eventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2cafe",
  "eventType": "alert.triggered",
  "occurredAt": "2026-06-01T10:55:00.123Z",
  "correlationId": "e2bbcb44-77db-4e26-bb2b-7cf0fbf3a44d",
  "source": "/campus/core-business/policy-engine",
  "data": {
    "alertId": "ALT-99823",
    "severity": "CRITICAL",
    "category": "FIRE_ALARM",
    "locationId": "hallway-floor-2-building-a",
    "description": "Phát hiện khói vượt ngưỡng 85% tại hành lang tầng 2 nhà A.",
    "channels": ["SMS", "MOBILE_PUSH"],
    "recipients": [
      {
        "role": "SECURITY_OFFICER",
        "department": "CAMPUS_SECURITY"
      }
    ]
  }
}
```

### 4.2. Event: `notification.broadcast.requested`
Được publish bởi Core Business khi cần gửi thông báo hành chính hàng loạt.

```json
{
  "eventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2caff",
  "eventType": "notification.broadcast.requested",
  "occurredAt": "2026-06-01T10:56:00.000Z",
  "correlationId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2cafe",
  "source": "/campus/core-business/admin-portal",
  "data": {
    "broadcastId": "BRC-0021",
    "title": "Thông báo khẩn cấp: Tạm dừng hoạt động Tòa nhà A",
    "body": "Do có sự cố kỹ thuật hệ thống điện, Tòa nhà A sẽ tạm ngừng hoạt động vào chiều ngày 01-06-2026. Sinh viên vui lòng di chuyển sang khu vực nhà B.",
    "channels": ["EMAIL", "MOBILE_PUSH"],
    "targetAudience": {
      "role": "STUDENT",
      "building": "BUILDING_A"
    }
  }
}
```

---

## 5. Ràng buộc kỹ thuật & Nghiệp vụ thống nhất

| Vấn đề đàm phán | Quyết định thống nhất | Rationale (Lý do kỹ thuật) |
| :--- | :--- | :--- |
| **Bảo mật kênh truyền** | Toàn bộ dữ liệu nhạy cảm của người nhận (như số điện thoại, email cụ thể) **không được truyền trực tiếp** trong payload. Thay vào đó chỉ truyền `role` hoặc `userId`. | Đảm bảo an toàn thông tin và tuân thủ luật bảo vệ dữ liệu cá nhân (GDPR/NDPR). Notification service tự tra cứu contact từ DB của mình. |
| **Độ ưu tiên gửi (Priority)** | Mức độ `severity: CRITICAL` bắt buộc dịch vụ Notification phải xử lý ưu tiên gửi SMS ngay lập tức (SLA < 3s). Mức độ `INFO` xử lý sau. | Bảo vệ tính mạng và tài sản trong trường hợp khẩn cấp, tối ưu chi phí SMS (SMS đắt hơn Push). |
| **Chống gửi trùng lặp** | Dựa vào `eventId` duy nhất. Notification phải lưu cache các ID đã gửi trong 12 tiếng để loại bỏ trùng lặp. | Tránh tình trạng gửi trùng nhiều tin nhắn gây hoang mang và lãng phí chi phí nhà mạng. |
| **Fallback Kênh** | Nếu gửi Push thất bại trong vòng 30 giây đối với tin khẩn cấp, Notification phải tự động fallback sang SMS. | Đảm bảo tin khẩn cấp chắc chắn tới tay người nhận. |

---

## 6. Các vấn đề chuyển tiếp sang Lab 03

1. **Định nghĩa chi tiết AsyncAPI**: Khai báo Channels, Message Payload Schema cho Kafka/RabbitMQ.
2. **Cấu hình DLQ và Retry**: Retry 5 lần với Exponential Backoff trước khi đưa vào hàng đợi lỗi `core.notification.dlq`.
3. **Cơ chế Report Trạng thái**: Thống nhất event phản hồi `notification.status.updated` từ Notification bắn ngược lại Core để cập nhật log.
