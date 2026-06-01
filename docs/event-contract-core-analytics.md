# Event Contract sơ bộ — Pair 08: Core Business ↔ Analytics

> **Lưu ý**: Bản hợp đồng sự kiện (Event Contract) này dùng để ghi nhận các thỏa thuận kỹ thuật ban đầu giữa nhóm **Core Business (A6)** và **Analytics (A8/A5)** cho luồng tích hợp bất đồng bộ qua Queue (Queue async). Đặc tả chi tiết bằng AsyncAPI sẽ được thực hiện chi tiết trong Lab 03.

---

## 1. Thông tin Dependency

- **Cặp tích hợp**: Pair 08 — Core Business (A6/B6) ↔ Analytics (A5/B5 trong pairing-matrix.md)
- **Publisher (Producer)**: Core Business (A6)
- **Subscriber (Consumer)**: Analytics (A5)
- **Cơ chế truyền thông**: Queue async (Message Broker / Kafka hoặc RabbitMQ)
- **Ngày thống nhất**: 01-06-2026
- **Đại diện hai bên**:
  - Đại diện Core Business (A6): Nguyễn Thị Hồng Duyên
  - Đại diện Analytics (A5): 

---

## 2. Mục đích nghiệp vụ

Để hệ thống Smart Campus cải thiện hiệu suất và tối ưu hóa tài nguyên vận hành, dịch vụ **Analytics (A5)** cần thu thập dữ liệu thống kê từ **Core Business (A6)**. Các sự kiện này bao gồm toàn bộ kết quả thẩm định chính sách ra vào vật lý và trạng thái giải quyết các sự cố an ninh. Analytics sẽ tổng hợp các dữ liệu này thành các chỉ số vận hành quan trọng (KPIs) hiển thị trên Dashboard giám sát tổng của Ban quản lý Campus.

---

## 3. Danh sách Event & Topic dự kiến

| Event Name | Topic / Routing Key | Mô tả nghiệp vụ | Tần suất |
| :--- | :--- | :--- | :--- |
| `policy.evaluation.completed` | `core.campus.policy.evaluation` | Bắn đi khi Core Business hoàn thành thẩm định một yêu cầu quẹt thẻ ra/vào. | Rất cao (mỗi lượt quẹt thẻ) |
| `alert.resolved` | `core.campus.alert.resolved` | Bắn đi khi một sự cố khẩn cấp (như cảnh báo cháy, đột nhập) chính thức được xử lý xong. | Thấp (khi có sự cố kết thúc) |

---

## 4. Payload chi tiết và Cấu trúc dữ liệu mẫu

Hai bên thống nhất sử dụng chuẩn **CloudEvents v1.0** làm vỏ bọc ngoài.

### 4.1. Event: `policy.evaluation.completed`
Gửi dữ liệu lượt thẩm định thẻ phục vụ thống kê lưu lượng người và hiệu quả của Policy Engine.

```json
{
  "eventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2caf1",
  "eventType": "policy.evaluation.completed",
  "occurredAt": "2026-06-01T10:00:00.050Z",
  "correlationId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2cafe",
  "source": "/campus/core-business/policy-engine",
  "data": {
    "decisionId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2caff",
    "cardId": "CARD-123456",
    "gateId": "GATE-01",
    "direction": "IN",
    "evaluatedAt": "2026-06-01T10:00:00.005Z",
    "decisionTimeMs": 45,
    "isAllowed": true,
    "reasonCode": "ALLOWED",
    "appliedPolicyId": "POL-101"
  }
}
```

### 4.2. Event: `alert.resolved`
Gửi dữ liệu khi sự cố an ninh kết thúc, giúp Analytics đo lường thời gian khắc phục sự cố (MTTR - Mean Time to Resolution).

```json
{
  "eventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2caf2",
  "eventType": "alert.resolved",
  "occurredAt": "2026-06-01T11:15:00.000Z",
  "correlationId": "e2bbcb44-77db-4e26-bb2b-7cf0fbf3a44d",
  "source": "/campus/core-business/incident-manager",
  "data": {
    "alertId": "ALT-99823",
    "resolvedAt": "2026-06-01T11:14:55.000Z",
    "triggeredAt": "2026-06-01T10:55:00.000Z",
    "durationSeconds": 1195,
    "resolutionCategory": "SYSTEM_ACKNOWLEDGED",
    "resolvedBy": "operator-002",
    "finalStatus": "RESOLVED_FALSE_ALARM"
  }
}
```

---

## 5. Ràng buộc kỹ thuật & Nghiệp vụ thống nhất

| Vấn đề đàm phán | Quyết định thống nhất | Rationale (Lý do kỹ thuật) |
| :--- | :--- | :--- |
| **Bảo mật và Mã hóa dữ liệu** | Analytics chỉ thu thập để thống kê tổng hợp (Aggregation), nên **không được lưu trữ thông tin định danh cá nhân** (Mã thẻ `cardId` phải được hash hoặc anonymize tại Analytics sau khi phân tích). | Đảm bảo tính riêng tư của sinh viên và cán bộ nhà trường theo đúng quy chế bảo mật dữ liệu học đường. |
| **Phân tích thời gian phản hồi (Latency)** | Core Business đính kèm thuộc tính `decisionTimeMs` (thời gian tính toán policy của Core). | Giúp Analytics vẽ biểu đồ giám sát SLA kỹ thuật của cụm máy chủ Policy Engine realtime. |
| **Độ trễ truyền tin tối đa** | Sự kiện phân tích thống kê có độ ưu tiên thấp. Cho phép delay lên tới 10 phút trước khi Analytics tiêu thụ. | Không gây tranh chấp tài nguyên mạng hoặc làm nghẽn hàng đợi của các luồng nghiệp vụ khẩn cấp (như cảnh báo cháy). |
| **Cơ chế chống trùng lắp** | Sử dụng `eventId` để loại bỏ trùng lặp tin nhắn. | Đảm bảo tính chính xác của dữ liệu KPI thống kê (không bị tăng khống lượt quẹt thẻ). |

---

## 6. Các vấn đề chuyển tiếp sang Lab 03

1. **Kafka Partitioning**: Thỏa thuận về cách chọn Partition Key. Thống nhất chọn `gateId` làm Partition Key cho `policy.evaluation.completed` để phân tích lưu lượng theo từng cổng cụ thể.
2. **Kế hoạch Data Retention**: Thống nhất thời gian lưu trữ dữ liệu thô tại Analytics là 30 ngày, sau đó nén và đưa vào Cold Storage/Data Lake phục vụ phân tích dài hạn (BigData).
