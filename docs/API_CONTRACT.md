# Hợp đồng API đồng bộ Backend – Frontend – Database

## 1. Quy ước chung

- Base URL local: `http://localhost:3000/api`.
- JSON request/response; header xác thực: `Authorization: Bearer <token>`.
- JSON dùng camelCase; thời gian dùng ISO 8601 có timezone, ví dụ `2026-09-28T08:00:00+07:00`.
- Thành công: `200` đọc/sửa, `201` tạo mới, `204` khi không có body.
- Lỗi: `400` dữ liệu sai, `401` chưa đăng nhập, `403` sai quyền, `404` không có dữ liệu, `409` xung đột/trạng thái sai.

Mẫu lỗi:

```json
{
  "error": "SLOT_UNAVAILABLE",
  "message": "Khung giờ của bác sĩ đã được đặt.",
  "details": {}
}
```

## 2. Xác thực và tài khoản

### `POST /auth/register` — công khai

```json
{
  "email": "patient@example.com",
  "password": "Password123",
  "fullName": "Nguyễn Văn A",
  "phone": "0901234567",
  "dateOfBirth": "2000-01-20",
  "address": "TP.HCM"
}
```

Trả `201`:

```json
{
  "user": { "id": 4, "email": "patient@example.com", "fullName": "Nguyễn Văn A", "role": "PATIENT" },
  "token": "jwt"
}
```

### `POST /auth/login` — công khai

Request: `{ "email": "...", "password": "..." }`.

Response giống register. Frontend lưu token cho phiên hiện tại; không tự tạo/sửa role.

### `GET /auth/me` — đã đăng nhập

Trả người dùng hiện tại để khôi phục phiên và dựng menu theo role.

## 3. Bệnh nhân và bác sĩ

| Method | Endpoint | Quyền | Mục đích |
|---|---|---|---|
| GET | `/patients/me` | PATIENT | Hồ sơ của chính mình |
| PATCH | `/patients/me` | PATIENT | Sửa thông tin liên hệ/y tế cho phép |
| GET | `/patients?q=` | ADMIN, RECEPTIONIST, DOCTOR | Tìm bệnh nhân theo tên/email/điện thoại |
| GET | `/doctors` | Tất cả role | Danh sách bác sĩ đang hoạt động |
| GET | `/doctors/:doctorId/availability?date=YYYY-MM-DD` | Đã đăng nhập | Slot trống trong ngày |

Mẫu slot:

```json
{
  "doctorId": 1,
  "date": "2026-09-29",
  "slots": [
    { "startAt": "2026-09-29T01:00:00.000Z", "endAt": "2026-09-29T01:30:00.000Z" }
  ]
}
```

## 4. Lịch hẹn — yêu cầu 2, 3, 4 và 12

### `GET /appointments?date=&status=`

- PATIENT chỉ nhận lịch của mình; DOCTOR chỉ lịch được giao; RECEPTIONIST/ADMIN nhận theo bộ lọc.
- Response: `{ "appointments": [...] }`.

### `POST /appointments`

PATIENT không gửi `patientId`; backend suy ra từ JWT. RECEPTIONIST/ADMIN bắt buộc gửi `patientId` khi đặt thay.

```json
{
  "doctorId": 1,
  "patientId": 5,
  "startAt": "2026-09-29T08:00:00+07:00",
  "endAt": "2026-09-29T08:30:00+07:00",
  "reason": "Đau răng hàm dưới"
}
```

Trả `201`: `{ "appointment": { "id": 10, "status": "PENDING", ... } }`.

### `PATCH /appointments/:id/status`

```json
{ "status": "CONFIRMED", "cancellationReason": null }
```

Ma trận chuyển trạng thái:

| Role | Từ | Sang |
|---|---|---|
| PATIENT | PENDING/CONFIRMED | CANCELLED (cần lý do) |
| RECEPTIONIST | PENDING | CONFIRMED/CANCELLED |
| RECEPTIONIST | CONFIRMED | CHECKED_IN/NO_SHOW/CANCELLED |
| DOCTOR được giao | CHECKED_IN | IN_PROGRESS |
| DOCTOR được giao | IN_PROGRESS | COMPLETED |
| ADMIN | Các bước quản trị hợp lệ | Theo cùng máy trạng thái |

### `POST /treatments/plans/:planId/follow-ups`

DOCTOR điều trị hoặc RECEPTIONIST:

```json
{
  "startAt": "2026-10-15T09:00:00+07:00",
  "endAt": "2026-10-15T09:30:00+07:00",
  "reason": "Tái khám sau điều trị"
}
```

Backend lấy patient/doctor từ plan, kiểm tra slot và ghi `parentPlanId`.

## 5. Khám và hồ sơ — yêu cầu 5, 6

### `POST /treatments/examinations` — DOCTOR được giao

```json
{
  "appointmentId": 10,
  "symptoms": "Ê buốt khi uống lạnh",
  "diagnosis": "Sâu răng 36",
  "notes": "Chưa có dấu hiệu viêm tủy"
}
```

Điều kiện: appointment là `IN_PROGRESS`; một appointment chỉ tạo một record. Trả `201` với `recordId`.

### `PATCH /treatments/records/:recordId`

DOCTOR sở hữu hồ sơ cập nhật ghi chú/trạng thái theo quy tắc; không đổi patient/doctor/appointment.

## 6. Dịch vụ, kế hoạch, báo giá — yêu cầu 7, 8, 9

| Method | Endpoint | Quyền | Mục đích |
|---|---|---|---|
| GET | `/services` | Tất cả role | Danh mục dịch vụ đang hoạt động |
| POST | `/services` | ADMIN | Tạo dịch vụ |
| PATCH | `/services/:id` | ADMIN | Đổi tên/giá/active |

Tạo/sửa dịch vụ:

```json
{ "code": "TRAM", "name": "Trám răng thẩm mỹ", "unitPrice": 500000, "active": true }
```

### `POST /treatments/:recordId/plans` — DOCTOR sở hữu hồ sơ

```json
{
  "discountPercent": 10,
  "notes": "Điều trị trong hai buổi",
  "items": [
    { "serviceId": 3, "quantity": 2, "toothArea": "36, 37", "notes": "Trám composite" }
  ]
}
```

Response phải trả dữ liệu server đã tính:

```json
{
  "plan": { "id": 8, "status": "PENDING_PATIENT", "subtotal": 1000000, "discountPercent": 10, "total": 900000 },
  "items": [{ "serviceId": 3, "quantity": 2, "unitPrice": 500000, "lineTotal": 1000000 }]
}
```

Frontend chỉ hiển thị tổng từ response, không dùng tổng tự tính để thanh toán.

## 7. Xác nhận — yêu cầu 10

### `PATCH /treatments/plans/:planId/decision` — PATIENT sở hữu plan

```json
{ "decision": "ACCEPTED" }
```

`decision` chỉ là `ACCEPTED` hoặc `REJECTED`; plan phải đang `PENDING_PATIENT`.

## 8. Thanh toán — yêu cầu 11

### `GET /payments/plan/:planId`

RECEPTIONIST/ADMIN hoặc PATIENT sở hữu plan. Response:

```json
{
  "total": 900000,
  "paid": 300000,
  "remaining": 600000,
  "payments": []
}
```

### `POST /payments`

```json
{ "planId": 8, "amount": 600000, "method": "TRANSFER", "transactionRef": "BANK-123" }
```

Backend kiểm tra quyền sở hữu, trạng thái và số dư trong transaction. Trả `201` với `remaining` và `planStatus`.

## 9. Lịch sử — yêu cầu 13

### `GET /treatments/history/:patientId?page=1&limit=20`

- PATIENT: `patientId` phải là hồ sơ của chính mình.
- DOCTOR: chỉ bệnh nhân có lịch/hồ sơ với mình.
- RECEPTIONIST: dữ liệu hành chính cần thiết; tránh trả ghi chú y khoa nhạy cảm nếu không cần.
- ADMIN: toàn bộ.

Response gợi ý:

```json
{
  "items": [
    {
      "recordId": 2,
      "appointmentId": 10,
      "examinedAt": "2026-09-29T02:00:00.000Z",
      "doctorName": "BS. Nguyễn Minh",
      "diagnosis": "Sâu răng 36",
      "plans": []
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 1
}
```

## 10. Mapping màn hình frontend

| Màn hình | API chính |
|---|---|
| Đăng ký/đăng nhập | `POST /auth/register`, `POST /auth/login`, `GET /auth/me` |
| Đặt lịch | `GET /doctors`, availability, `POST /appointments` |
| Lịch của tôi/lịch bác sĩ | `GET /appointments` |
| Quầy tiếp nhận | tìm bệnh nhân, GET/PATCH appointments |
| Khám | PATCH status, POST examinations |
| Kế hoạch/báo giá | GET services, POST plans |
| Bệnh nhân xác nhận | PATCH decision |
| Thanh toán | GET/POST payments |
| Tái khám | POST follow-ups |
| Lịch sử | GET history |

## 11. Checklist chống lệch contract

- Backend và frontend dùng đúng `fullName`, không xen kẽ `name`/`full_name` trong JSON.
- Dùng thống nhất `startAt`, `endAt`, không dùng `date` + `time` ở request đặt lịch.
- Role/status viết hoa đúng enum.
- Frontend chỉ báo thành công khi HTTP response thành công; không nuốt lỗi rồi chuyển sang dữ liệu giả.
- Demo mode phải bật rõ bằng `VITE_DEMO_MODE=true`; API mode không tự fallback.

