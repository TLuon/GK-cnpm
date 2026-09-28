# Hướng dẫn nghiệp vụ và phân quyền

## 1. Phạm vi hệ thống

Hệ thống quản lý phòng khám nha khoa hỗ trợ 13 chức năng: đăng ký bệnh nhân, đặt lịch, kiểm tra lịch bác sĩ, tiếp nhận, khám, lập hồ sơ điều trị, chỉ định dịch vụ, lập kế hoạch điều trị, báo giá, xác nhận điều trị, thanh toán, đặt lịch tái khám và xem lịch sử điều trị.

## 2. Vai trò và quyền

| Chức năng | Bệnh nhân | Lễ tân | Bác sĩ | Quản trị viên |
|---|:---:|:---:|:---:|:---:|
| Đăng ký/đăng nhập | Chính mình | Tạo bệnh nhân | Đăng nhập | Quản lý tài khoản |
| Xem lịch bác sĩ/slot trống | Có | Có | Lịch của mình | Có |
| Đặt/hủy lịch | Chính mình | Đặt thay | Không | Có |
| Tiếp nhận/check-in | Không | Có | Xem | Có |
| Khám, chẩn đoán | Xem sau khi lưu | Không | Có, với ca được giao | Có |
| Chỉ định dịch vụ/kế hoạch | Xem | Không | Có | Có |
| Quản lý danh mục/đơn giá | Không | Xem | Xem | Có |
| Xác nhận điều trị | Chính hồ sơ của mình | Ghi nhận tại quầy | Không xác nhận thay | Có khi có ủy quyền |
| Thu/ghi nhận thanh toán | Xem biên nhận | Có | Không | Có |
| Lịch sử điều trị | Của mình | Khi phục vụ nghiệp vụ | Bệnh nhân mình điều trị | Có |

Frontend chỉ ẩn/hiện tính năng để cải thiện trải nghiệm. Mọi quyền bắt buộc được kiểm tra lại tại API dựa trên token; không tin `role`, `patientId` hoặc `doctorId` do trình duyệt tự gửi.

## 3. Luồng nghiệp vụ chuẩn

1. **Đăng ký bệnh nhân:** email/số điện thoại không trùng; mật khẩu được băm; tài khoản mới chỉ mang vai trò `PATIENT`.
2. **Đặt lịch:** chọn bác sĩ và slot tương lai. API kiểm tra lịch làm việc, thời lượng, lịch trùng của bác sĩ và bệnh nhân trong transaction.
3. **Tiếp nhận:** lễ tân chỉ check-in lịch `CONFIRMED` đúng ngày. Lịch bị hủy hoặc đã hoàn tất không thể tiếp nhận lại.
4. **Khám:** bác sĩ được phân công chuyển `CHECKED_IN -> IN_PROGRESS`, ghi triệu chứng, chẩn đoán và ghi chú lâm sàng.
5. **Hồ sơ/kế hoạch:** mỗi kế hoạch thuộc một lần khám; gồm nhiều dịch vụ, răng/vùng điều trị, số lượng, đơn giá chốt tại thời điểm lập và ghi chú.
6. **Báo giá:** tổng trước giảm giá bằng tổng `đơn giá chốt × số lượng`; giảm giá không âm và không lớn hơn tổng; tổng thanh toán không âm. Không lấy lại giá hiện tại để sửa báo giá cũ.
7. **Xác nhận:** bệnh nhân sở hữu kế hoạch có thể `APPROVE` hoặc `REJECT`. Kế hoạch đã bắt đầu hoặc hoàn tất không được xác nhận lại.
8. **Thanh toán:** chỉ thu cho kế hoạch đã đồng ý; số tiền dương, tổng các giao dịch thành công không vượt tổng báo giá. Trạng thái là `UNPAID`, `PARTIALLY_PAID`, `PAID` theo số dư.
9. **Hoàn tất và tái khám:** bác sĩ hoàn tất điều trị, sau đó bệnh nhân/lễ tân có thể tạo lịch tái khám liên kết hồ sơ cũ nhưng vẫn phải qua kiểm tra slot.
10. **Lịch sử:** là dữ liệu chỉ đọc tổng hợp từ lịch hẹn, lần khám, kế hoạch, dịch vụ và thanh toán; lọc theo quyền sở hữu/phạm vi công việc.

## 4. Máy trạng thái

### Lịch hẹn

```text
PENDING -> CONFIRMED -> CHECKED_IN -> IN_PROGRESS -> COMPLETED
    |           |             |
    +-----------+-------------+-> CANCELLED
                +----------------> NO_SHOW
```

- Bệnh nhân chỉ được hủy lịch của mình trước ngưỡng thời gian cấu hình.
- Lễ tân xác nhận, check-in, đánh dấu vắng; bác sĩ bắt đầu và hoàn tất ca khám.
- API từ chối mọi bước nhảy trạng thái không nằm trong danh sách cho phép.

### Kế hoạch điều trị

```text
DRAFT -> QUOTED -> APPROVED -> IN_PROGRESS -> COMPLETED
                    |              |
QUOTED -> REJECTED  +--------------+-> CANCELLED
```

## 5. Công thức tính chi phí

Với mỗi dòng dịch vụ `i`:

```text
lineTotal_i = unitPrice_i * quantity_i
subTotal    = sum(lineTotal_i)
discount    = fixedDiscount + percentDiscount * subTotal / 100
grandTotal  = max(0, subTotal - discount)
balance     = grandTotal - sum(successfulPayments)
```

Tiền lưu bằng số nguyên VND để tránh sai số dấu phẩy động. Phần trăm giảm giá nằm trong `0..100`; mọi thay đổi giá hoặc giảm giá phải được server tính lại, không nhận `grandTotal` từ frontend.

## 6. An toàn và toàn vẹn dữ liệu

- Mật khẩu băm bằng bcrypt; JWT có hạn dùng và secret lấy từ biến môi trường.
- Trả `401` khi chưa xác thực, `403` khi sai quyền, `404` khi không tồn tại, `409` khi trùng lịch/trùng dữ liệu.
- Dùng prepared statement, validation đầu vào và transaction cho đặt lịch, xác nhận kế hoạch, thanh toán.
- Không trả `password_hash`; không cho bệnh nhân đọc hồ sơ người khác.
- Lưu `created_at`, `updated_at` và người thực hiện cho các thao tác quan trọng.

## 7. Chạy và trình diễn

1. Cài dependencies backend và frontend theo README tương ứng.
2. Tạo `.env` từ `.env.example`, thay `JWT_SECRET` khi triển khai thật.
3. Khởi động API, sau đó chạy Vite frontend.
4. Demo lần lượt bằng bốn vai trò: bệnh nhân đặt lịch, lễ tân tiếp nhận, bác sĩ lập kế hoạch, bệnh nhân xác nhận và lễ tân thu tiền.

