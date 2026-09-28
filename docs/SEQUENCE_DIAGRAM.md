# Sequence Diagram - Đặt lịch khám nha khoa

Sơ đồ mô tả luồng đặt lịch có kiểm tra xác thực, quyền truy cập, lịch làm việc và chống trùng lịch ở phía máy chủ.

```mermaid
sequenceDiagram
    autonumber
    actor BN as Bệnh nhân
    participant UI as React Web App
    participant API as Appointment API
    participant Auth as JWT/RBAC Middleware
    participant DB as SQLite Database

    BN->>UI: Chọn bác sĩ, ngày và giờ khám
    UI->>API: GET /api/doctors/:id/slots?date=...
    API->>DB: Đọc lịch làm việc và lịch hẹn hiện có
    DB-->>API: Khung giờ còn trống
    API-->>UI: Danh sách khung giờ khả dụng
    BN->>UI: Xác nhận đặt lịch + lý do khám
    UI->>API: POST /api/appointments (Bearer token)
    API->>Auth: Xác thực token và quyền PATIENT
    alt Chưa đăng nhập hoặc sai quyền
        Auth-->>UI: 401/403
    else Hợp lệ
        API->>DB: BEGIN IMMEDIATE TRANSACTION
        API->>DB: Kiểm tra bệnh nhân, bác sĩ, lịch làm việc và slot
        alt Dữ liệu không hợp lệ
            DB-->>API: Không hợp lệ
            API->>DB: ROLLBACK
            API-->>UI: 400 - Chi tiết lỗi
        else Slot đã được giữ/đặt
            DB-->>API: Xung đột unique doctor/time
            API->>DB: ROLLBACK
            API-->>UI: 409 - Khung giờ không còn trống
        else Slot hợp lệ
            API->>DB: INSERT appointment (PENDING)
            API->>DB: COMMIT
            DB-->>API: Mã lịch hẹn
            API-->>UI: 201 - Đặt lịch thành công
            UI-->>BN: Hiển thị mã và trạng thái lịch hẹn
        end
    end
```

## Quy tắc quan trọng

- Bệnh nhân chỉ được đặt lịch cho chính mình; lễ tân hoặc quản trị viên có thể đặt thay.
- Thời điểm khám phải ở tương lai, nằm trong lịch làm việc của bác sĩ và đúng độ dài slot.
- Cơ sở dữ liệu có ràng buộc duy nhất theo bác sĩ và thời điểm để chặn hai yêu cầu đồng thời.
- Trạng thái hợp lệ: `PENDING -> CONFIRMED -> CHECKED_IN -> IN_PROGRESS -> COMPLETED`; có thể chuyển sang `CANCELLED` hoặc `NO_SHOW` theo quyền và thời điểm.

