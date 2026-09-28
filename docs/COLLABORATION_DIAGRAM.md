# Collaboration Diagram - Đặt lịch khám nha khoa

Sơ đồ giao tiếp (communication/collaboration) tập trung vào các đối tượng và thông điệp đánh số trong ca sử dụng đặt lịch.

```mermaid
flowchart LR
    P["Bệnh nhân"]
    UI["React Appointment Form"]
    AUTH["Auth/RBAC"]
    CTRL["Appointment Controller"]
    SVC["Appointment Service"]
    DB[("SQLite")]

    P -->|"1: chọn bác sĩ/ngày"| UI
    UI -->|"1.1: yêu cầu slot trống"| CTRL
    CTRL -->|"1.2: truy vấn lịch"| SVC
    SVC -->|"1.3: đọc lịch làm việc + lịch hẹn"| DB
    DB -->|"1.4: slot khả dụng"| SVC
    SVC -->|"1.5"| CTRL
    CTRL -->|"1.6: hiển thị slot"| UI
    P -->|"2: xác nhận thông tin"| UI
    UI -->|"2.1: POST lịch hẹn + JWT"| AUTH
    AUTH -->|"2.2: danh tính và vai trò"| CTRL
    CTRL -->|"2.3: tạo lịch"| SVC
    SVC -->|"2.4: transaction kiểm tra và ghi"| DB
    DB -->|"2.5: appointment hoặc conflict"| SVC
    SVC -->|"2.6: kết quả nghiệp vụ"| CTRL
    CTRL -->|"2.7: HTTP 201/400/409"| UI
    UI -->|"2.8: thông báo kết quả"| P
```

## Trách nhiệm đối tượng

| Đối tượng | Trách nhiệm |
|---|---|
| React Appointment Form | Thu thập dữ liệu, kiểm tra cơ bản và hiển thị phản hồi |
| Auth/RBAC | Xác thực JWT và chặn vai trò không được phép |
| Appointment Controller | Chuẩn hóa request/response HTTP |
| Appointment Service | Áp dụng quy tắc lịch làm việc, thời gian và chuyển trạng thái |
| SQLite | Lưu bền vững và đảm bảo không trùng lịch bằng transaction/constraint |

