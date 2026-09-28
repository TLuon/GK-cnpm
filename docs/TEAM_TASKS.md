# Phân công chi tiết cho ba thành viên

## 1. Nguyên tắc chia việc

- Người 1 sở hữu nền tảng, xác thực, bệnh nhân, bác sĩ và lịch hẹn.
- Người 2 sở hữu nghiệp vụ điều trị, dịch vụ, báo giá, thanh toán và lịch sử.
- Người 3 sở hữu toàn bộ React UI và chỉ tích hợp qua hợp đồng API.
- Hai backend cùng quản lý database nhưng không tự ý đổi bảng/API của nhau. Mọi thay đổi chung phải ghi vào `DATABASE_DESIGN.md` và `API_CONTRACT.md`.

## 2. Người 1 — Backend + Database: nền tảng và lịch hẹn

### Phạm vi yêu cầu

Chịu trách nhiệm yêu cầu 1–4 và nền tảng dùng chung cho yêu cầu 12:

1. Đăng ký bệnh nhân.
2. Đặt lịch.
3. Kiểm tra lịch bác sĩ.
4. Tiếp nhận.

### Việc phải làm

1. Khởi tạo Node.js/Express, middleware JSON, CORS, `.env` và error handler chung.
2. Tạo schema/migration cho `users`, `patients`, `doctors`, `doctor_schedules`, `appointments`.
3. Viết đăng ký/đăng nhập, bcrypt, JWT và middleware `authenticate`, `authorize`.
4. Tạo seed tối thiểu cho admin, lễ tân, bác sĩ và lịch làm việc.
5. Viết API hồ sơ bệnh nhân, danh sách bác sĩ và slot trống.
6. Viết đặt lịch trong transaction; chặn trùng bác sĩ và bệnh nhân.
7. Viết máy trạng thái lịch hẹn và kiểm tra người sở hữu/bác sĩ được giao.
8. Ghi audit check-in và lý do hủy.
9. Cung cấp hàm/service kiểm tra slot để Người 2 dùng cho tái khám.
10. Viết test API cho đăng ký, login, slot, đặt lịch, conflict, xác nhận và check-in.

### API bàn giao

- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/auth/me`
- `GET|PATCH /api/patients/me`
- `GET /api/patients?q=`
- `GET /api/doctors`
- `GET /api/doctors/:id/availability?date=`
- `GET|POST /api/appointments`
- `PATCH /api/appointments/:id/status`

### Điều kiện nghiệm thu

- JWT không chứa/thay thế quyền từ request; role đọc lại từ database.
- Bệnh nhân không thể đặt cho người khác hoặc xem lịch người khác.
- Hai request đồng thời không tạo hai lịch trùng slot.
- Không cho nhảy `PENDING -> COMPLETED` hoặc check-in lịch đã hủy.
- Tất cả lỗi đúng HTTP status và cấu trúc trong `API_CONTRACT.md`.

## 3. Người 2 — Backend + Database: điều trị và tài chính

### Phạm vi yêu cầu

Chịu trách nhiệm yêu cầu 5–13, dùng lịch hẹn và user từ Người 1:

5. Khám.
6. Lập hồ sơ điều trị.
7. Chỉ định dịch vụ.
8. Lập kế hoạch điều trị.
9. Báo giá.
10. Xác nhận điều trị.
11. Thanh toán.
12. Đặt lịch tái khám.
13. Xem lịch sử điều trị.

### Việc phải làm

1. Tạo schema/migration cho `dental_services`, `treatment_records`, `treatment_plans`, `treatment_plan_items`, `payments`; bổ sung `parent_plan_id` vào appointment.
2. Viết CRUD dịch vụ; chỉ admin thay giá/active.
3. Viết API khám, kiểm tra appointment `IN_PROGRESS` và bác sĩ được giao.
4. Viết tạo kế hoạch + items trong một transaction.
5. Tính giá bằng service thuần, dùng integer VND; snapshot đơn giá vào item.
6. Viết xác nhận/từ chối, chỉ bệnh nhân sở hữu plan và chỉ quyết định một lần.
7. Viết thanh toán một phần/toàn phần, chặn trả vượt số dư; transaction payment + plan.
8. Viết tái khám bằng service kiểm tra slot của Người 1, liên kết plan gốc.
9. Viết lịch sử có scope theo role và phân trang.
10. Viết test unit cho calculator và integration cho toàn luồng khám–plan–decision–payment.

### API bàn giao

- `GET|POST /api/services`
- `PATCH /api/services/:id`
- `POST /api/treatments/examinations`
- `PATCH /api/treatments/records/:id`
- `POST /api/treatments/:recordId/plans`
- `PATCH /api/treatments/plans/:planId/decision`
- `POST /api/payments`
- `GET /api/payments/plan/:planId`
- `POST /api/treatments/plans/:planId/follow-ups`
- `GET /api/treatments/history/:patientId`

### Điều kiện nghiệm thu

- Bác sĩ A không sửa hồ sơ của bác sĩ B.
- Bệnh nhân A không xem/duyệt/trả tiền plan của bệnh nhân B.
- Đổi giá dịch vụ không làm đổi báo giá cũ.
- Backend bỏ qua mọi subtotal/total do frontend gửi.
- Hai lần thanh toán đồng thời không làm tổng đã trả vượt total.
- Lịch sử không lộ dữ liệu ngoài phạm vi role.

## 4. Người 3 — Frontend React

### Phạm vi

Tạo giao diện cho đủ 13 yêu cầu và bốn role; không trực tiếp truy cập database.

### Việc phải làm

1. Khởi tạo React/Vite, router/layout, theme responsive, API client và auth context.
2. Tạo màn đăng ký/đăng nhập; khôi phục phiên bằng `/auth/me`.
3. Menu theo role nhưng vẫn xử lý `401/403` từ server.
4. Màn bệnh nhân: dashboard, chọn bác sĩ/ngày/slot, đặt/hủy lịch, xem kế hoạch, xác nhận, thanh toán, tái khám và lịch sử.
5. Màn lễ tân: tìm bệnh nhân, đặt thay, danh sách theo ngày, xác nhận, check-in, no-show, thu tiền, tái khám.
6. Màn bác sĩ: lịch hôm nay, bắt đầu khám, form chẩn đoán, chọn dịch vụ, lập kế hoạch và xem lịch sử liên quan.
7. Màn admin: tài khoản/bác sĩ, lịch làm việc, dịch vụ và đơn giá.
8. Tạo component dùng chung: `StatusBadge`, `Money`, `DateTime`, `ConfirmDialog`, `ErrorBanner`, `Loading`, `EmptyState`.
9. Validation client khớp server, nhưng không thay thế validation backend.
10. Hiển thị lỗi theo `response.error/message`; không báo thành công nếu fetch thất bại.
11. Format VND bằng `Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' })`.
12. Build production và test ít nhất một luồng cho mỗi role.

### Cách gọi API thống nhất

```js
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:3000/api';

export async function api(path, options = {}) {
  const token = sessionStorage.getItem('token');
  const response = await fetch(`${API_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers
    }
  });
  const body = response.status === 204 ? null : await response.json();
  if (!response.ok) throw Object.assign(new Error(body?.message || 'Có lỗi xảy ra'), { status: response.status, body });
  return body;
}
```

Ví dụ đặt lịch:

```js
await api('/appointments', {
  method: 'POST',
  body: JSON.stringify({ doctorId, startAt, endAt, reason })
});
```

### Điều kiện nghiệm thu

- Không gửi `role` lúc đăng ký, không gửi `patientId` khi bệnh nhân đặt lịch cho mình.
- Không tự ghép `date/time` trái contract; gửi ISO `startAt/endAt`.
- Không dùng dữ liệu giả khi `VITE_DEMO_MODE` không bật.
- Mọi form có disabled/loading, lỗi field/server, empty state và thông báo thành công sau response thật.
- Responsive tốt ở điện thoại và desktop; build Vite không lỗi.

## 5. Điểm tích hợp giữa ba người

### Backend 1 bàn giao cho Backend 2

- Schema user/patient/doctor/appointment và ID thật.
- `authenticate/authorize` cùng shape `req.user = { id, role, ... }`.
- Service `checkDoctorAvailability(doctorId, startAt, endAt)` để tái sử dụng.
- Máy trạng thái lịch hẹn; Người 2 không cập nhật trực tiếp sai quy tắc.

### Hai backend bàn giao cho Frontend

- OpenAPI/Postman hoặc tối thiểu toàn bộ ví dụ trong `API_CONTRACT.md`.
- Tài khoản seed, base URL, mã lỗi và enum.
- Endpoint health `GET /api/health`.
- Xác nhận dữ liệu JSON dùng camelCase nhất quán.

### Frontend phản hồi cho backend

- Báo mismatch bằng ví dụ request/response cụ thể, không tự đổi tên field tạm thời.
- Không yêu cầu backend trả dữ liệu nhạy cảm chỉ để tiện hiển thị.
- Nếu cần endpoint tổng hợp dashboard, thống nhất contract trước khi viết.

## 6. Thứ tự ghép bài trong 60 phút

| Thời gian | Backend 1 | Backend 2 | Frontend |
|---|---|---|---|
| 0–10 phút | Scaffold, auth, schema nền | Schema điều trị, calculator | Scaffold, theme, auth UI |
| 10–30 phút | Doctor/slot/appointment | Exam/plan/services | Patient booking + dashboards |
| 30–45 phút | Status/check-in + tests | Decision/payment/history | Reception/doctor/treatment UI |
| 45–55 phút | Fix contract | Fix contract | Nối API thật, xử lý lỗi |
| 55–60 phút | Test smoke | Test smoke | Build và demo luồng chính |

## 7. Kịch bản demo chung

1. PATIENT đăng ký và đặt lịch.
2. RECEPTIONIST xác nhận rồi check-in.
3. DOCTOR bắt đầu khám, ghi chẩn đoán và lập kế hoạch.
4. PATIENT xem báo giá và đồng ý.
5. RECEPTIONIST ghi nhận hai lần thanh toán đến khi `PAID`.
6. RECEPTIONIST/DOCTOR đặt tái khám.
7. PATIENT mở lịch sử và thấy toàn bộ chuỗi dữ liệu.

