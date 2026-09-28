# Sequence Diagram — Đặt lịch khám nha khoa

## 1. Mục tiêu và phạm vi

Ca sử dụng mô tả bệnh nhân đã có tài khoản tìm slot và đặt lịch cho chính mình. Sơ đồ thể hiện đầy đủ frontend, xác thực/phân quyền, controller, service và database. Trường hợp lễ tân đặt thay dùng cùng service nhưng được phép gửi `patientId`.

### Tiền điều kiện

- Bệnh nhân có tài khoản đang hoạt động và đã đăng nhập.
- Bác sĩ đang hoạt động và có lịch làm việc.
- Frontend có JWT hợp lệ.

### Hậu điều kiện thành công

- Có một appointment `PENDING`, liên kết đúng patient/doctor.
- Slot không trùng với lịch hợp lệ của bác sĩ hoặc bệnh nhân.
- Response `201` có appointment để frontend hiển thị.

## 2. Sequence Diagram chi tiết

```mermaid
sequenceDiagram
    autonumber
    actor BN as Bệnh nhân
    participant UI as React Web App
    participant AUTH as Auth/RBAC Middleware
    participant AC as Appointment Controller
    participant AS as Appointment Service
    participant DB as SQLite Database

    BN->>UI: Mở màn hình đặt lịch
    UI->>AUTH: GET /api/doctors + Bearer JWT
    AUTH->>DB: Đọc user theo JWT.sub
    DB-->>AUTH: User active, role PATIENT
    AUTH->>AC: Request + req.user
    AC->>DB: Đọc danh sách bác sĩ active
    DB-->>AC: Danh sách bác sĩ
    AC-->>UI: 200 { doctors }

    BN->>UI: Chọn bác sĩ và ngày khám
    UI->>AUTH: GET /doctors/:id/availability?date=...
    AUTH->>AC: Request đã xác thực
    AC->>AS: getAvailableSlots(doctorId, date)
    AS->>DB: Đọc lịch làm việc bác sĩ
    DB-->>AS: startTime, endTime, slotMinutes
    AS->>DB: Đọc appointment chiếm chỗ trong ngày
    DB-->>AS: PENDING/CONFIRMED/CHECKED_IN/IN_PROGRESS
    AS->>AS: Sinh slot và loại slot trùng/quá khứ
    AS-->>AC: Danh sách slot trống
    AC-->>UI: 200 { doctorId, date, slots }
    UI-->>BN: Hiển thị các giờ có thể chọn

    BN->>UI: Chọn slot, nhập lý do và xác nhận
    UI->>UI: Kiểm tra doctorId, startAt, endAt, reason
    UI->>AUTH: POST /api/appointments + JWT + JSON

    alt Thiếu/sai/hết hạn token
        AUTH-->>UI: 401 AUTH_REQUIRED/INVALID_TOKEN
        UI-->>BN: Yêu cầu đăng nhập lại
    else Tài khoản không active hoặc sai role
        AUTH-->>UI: 401 hoặc 403 FORBIDDEN
        UI-->>BN: Thông báo không có quyền
    else Xác thực thành công
        AUTH->>AC: req.user và payload
        AC->>AC: Validate schema, parse ISO datetime
        alt Payload sai hoặc thời gian quá khứ
            AC-->>UI: 400 VALIDATION_ERROR
            UI-->>BN: Hiển thị lỗi đúng trường
        else Payload hợp lệ
            AC->>AS: createAppointment(req.user, input)
            AS->>DB: BEGIN IMMEDIATE TRANSACTION
            AS->>DB: Tìm patient bằng req.user.id
            DB-->>AS: patientId
            AS->>DB: Kiểm tra doctor active + lịch làm việc
            DB-->>AS: Doctor/schedule hoặc không tồn tại

            alt Bác sĩ không tồn tại/không làm giờ đó
                AS->>DB: ROLLBACK
                AS-->>AC: 404 DOCTOR_NOT_FOUND hoặc 400 OUTSIDE_WORKING_HOURS
                AC-->>UI: Error response
            else Bác sĩ hợp lệ
                AS->>DB: Kiểm tra overlap theo doctorId
                DB-->>AS: Conflict hoặc rỗng
                AS->>DB: Kiểm tra overlap theo patientId
                DB-->>AS: Conflict hoặc rỗng

                alt Bác sĩ đã có lịch
                    AS->>DB: ROLLBACK
                    AS-->>AC: 409 SLOT_UNAVAILABLE
                    AC-->>UI: 409 + message
                    UI->>UI: Tải lại danh sách slot
                    UI-->>BN: Yêu cầu chọn giờ khác
                else Bệnh nhân có lịch trùng
                    AS->>DB: ROLLBACK
                    AS-->>AC: 409 PATIENT_TIME_CONFLICT
                    AC-->>UI: 409 + message
                    UI-->>BN: Thông báo lịch cá nhân bị trùng
                else Không có xung đột
                    AS->>DB: INSERT appointment status=PENDING
                    DB-->>AS: appointmentId
                    AS->>DB: COMMIT
                    AS->>DB: SELECT appointment + patient + doctor
                    DB-->>AS: Appointment đầy đủ
                    AS-->>AC: Appointment
                    AC-->>UI: 201 { appointment }
                    UI-->>BN: Hiển thị mã lịch và trạng thái chờ xác nhận
                end
            end
        end
    end
```

## 3. Thông điệp và API tương ứng

| Bước | Thông điệp | Endpoint/Thao tác |
|---:|---|---|
| 1–6 | Tải bác sĩ | `GET /api/doctors` |
| 7–16 | Tìm slot | `GET /api/doctors/:doctorId/availability?date=` |
| 17–20 | Nhập/xác nhận | Validation phía React |
| 21–28 | Xác thực và validate | JWT middleware + schema |
| 29–44 | Kiểm tra và tạo lịch | Transaction trong Appointment Service |
| 45–50 | Trả kết quả | `201`, `400`, `401`, `403`, `404`, `409` |

## 4. Quy tắc cài đặt bắt buộc

- Frontend không gửi `patientId` khi PATIENT đặt cho mình; backend suy ra từ JWT.
- Availability chỉ giúp UX, không thay thế kiểm tra xung đột lúc insert.
- Query overlap chuẩn: lịch cũ có `start_at < newEnd` và `end_at > newStart`.
- Chỉ các trạng thái chưa hủy/vắng mới chiếm slot.
- Transaction phải đủ ngắn và rollback ở mọi lỗi.
- Response không chứa dữ liệu nhạy cảm; thời gian trả ISO UTC.

## 5. Sequence rút gọn cho lễ tân đặt thay

```mermaid
sequenceDiagram
    actor LT as Lễ tân
    participant UI as Reception UI
    participant API as Appointment API
    participant DB as Database
    LT->>UI: Tìm và chọn bệnh nhân
    UI->>API: GET /patients?q=...
    API-->>UI: Danh sách bệnh nhân theo quyền
    LT->>UI: Chọn doctor/slot và xác nhận
    UI->>API: POST /appointments có patientId
    API->>DB: Transaction kiểm tra quyền + conflict
    alt Hợp lệ
        DB-->>API: appointment PENDING
        API-->>UI: 201 Created
    else Conflict
        API-->>UI: 409 Conflict
    end
```

