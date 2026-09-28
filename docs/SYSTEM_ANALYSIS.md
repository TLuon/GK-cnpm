# Phân tích hệ thống quản lý phòng khám nha khoa

## 1. Bài toán

Bệnh nhân đặt lịch khám; lễ tân kiểm tra và tiếp nhận; bác sĩ khám, lập hồ sơ và kế hoạch điều trị; bệnh nhân duyệt báo giá; hệ thống ghi nhận dịch vụ, thanh toán, tái khám và lịch sử. Hệ thống phải đảm bảo đúng quyền, không trùng lịch, không sửa sai trạng thái và không tính sai chi phí.

## 2. Tác nhân

| Tác nhân | Trách nhiệm chính |
|---|---|
| Bệnh nhân (`PATIENT`) | Đăng ký, đặt/hủy lịch của mình, xem kế hoạch, xác nhận điều trị, xem thanh toán và lịch sử của mình |
| Lễ tân (`RECEPTIONIST`) | Tra cứu bệnh nhân, đặt lịch thay, xác nhận lịch, check-in, ghi nhận thanh toán và đặt tái khám |
| Bác sĩ (`DOCTOR`) | Xem lịch được giao, bắt đầu khám, ghi chẩn đoán, chỉ định dịch vụ, lập kế hoạch và hoàn tất điều trị |
| Quản trị viên (`ADMIN`) | Quản lý tài khoản, bác sĩ, lịch làm việc, danh mục dịch vụ, giá và giám sát toàn hệ thống |
| Hệ thống | Xác thực, phân quyền, kiểm tra xung đột, tính chi phí, bảo toàn dữ liệu và lưu dấu thời gian |

## 3. Phân tích đủ 13 yêu cầu

### 1. Đăng ký bệnh nhân

- Nhập họ tên, email, số điện thoại, mật khẩu; ngày sinh và địa chỉ là tùy chọn.
- Email là duy nhất, chuẩn hóa chữ thường; mật khẩu tối thiểu 8 ký tự có chữ và số, chỉ lưu bản băm.
- Tài khoản tự đăng ký luôn có role `PATIENT`; client không được chọn role.
- Tạo `users` và `patients` trong cùng transaction để không có tài khoản mồ côi.

### 2. Đặt lịch

- Bệnh nhân đặt cho chính mình; lễ tân/admin có thể chọn bệnh nhân.
- Phải chọn bác sĩ, giờ bắt đầu, giờ kết thúc và lý do khám.
- Thời gian ở tương lai, nằm trong lịch làm việc, không vượt thời lượng cho phép.
- Transaction phải kiểm tra trùng lịch của cả bác sĩ lẫn bệnh nhân; trả `409 SLOT_UNAVAILABLE` khi xung đột.
- Lịch mới có trạng thái `PENDING`; lịch tái khám có liên kết đến kế hoạch điều trị gốc.

### 3. Kiểm tra lịch bác sĩ

- Tính slot từ lịch làm việc trừ lịch `PENDING`, `CONFIRMED`, `CHECKED_IN`, `IN_PROGRESS`.
- Không coi lịch `CANCELLED` hoặc `NO_SHOW` là đang chiếm slot.
- API trả slot theo ngày và bác sĩ; backend vẫn kiểm tra lại khi POST đặt lịch vì slot có thể vừa bị người khác lấy.

### 4. Tiếp nhận

- Lễ tân xác nhận `PENDING -> CONFIRMED`, sau đó check-in `CONFIRMED -> CHECKED_IN`.
- Chỉ tiếp nhận đúng lịch chưa hủy; ghi `checked_in_at` và `checked_in_by`.
- Lễ tân có thể đánh dấu `NO_SHOW` cho lịch đã xác nhận nhưng bệnh nhân không đến.

### 5. Khám

- Chỉ bác sĩ được phân công mới khám lịch đó.
- Bác sĩ chuyển `CHECKED_IN -> IN_PROGRESS`, nhập triệu chứng, chẩn đoán và ghi chú.
- Một lịch khám chỉ có một hồ sơ khám chính; dữ liệu y khoa không được cho bệnh nhân khác hoặc bác sĩ không liên quan truy cập.

### 6. Lập hồ sơ điều trị

- Hồ sơ liên kết duy nhất với lịch hẹn, bệnh nhân và bác sĩ.
- Lưu chẩn đoán, ghi chú, ngày tạo và người tạo; không xóa cứng hồ sơ đã phát sinh kế hoạch/thanh toán.
- Trạng thái hồ sơ: `EXAMINED`, `IN_TREATMENT`, `COMPLETED`.

### 7. Chỉ định dịch vụ

- Bác sĩ chọn dịch vụ đang hoạt động, số lượng, răng/vùng điều trị và ghi chú.
- Đơn giá phải lấy từ database tại thời điểm lập kế hoạch và chụp lại vào dòng chi tiết.
- Chỉ admin tạo/sửa/ẩn dịch vụ; không xóa dịch vụ đã được sử dụng.

### 8. Lập kế hoạch điều trị

- Một hồ sơ có thể có nhiều kế hoạch/phiên bản nhưng chỉ kế hoạch hợp lệ mới được thực hiện.
- Kế hoạch gồm các dòng dịch vụ, giảm giá, ghi chú và trạng thái.
- Bác sĩ chỉ lập kế hoạch cho bệnh nhân mình điều trị.

### 9. Báo giá

- `lineTotal = unitPrice × quantity`.
- `subtotal = tổng lineTotal`; `discount = subtotal × discountPercent / 100`; `total = subtotal - discount`.
- `discountPercent` trong `0..100`, số lượng là số nguyên dương, tiền là số nguyên VND.
- Server tự tính; mọi `total` gửi từ frontend phải bị bỏ qua.

### 10. Xác nhận điều trị

- Chỉ bệnh nhân sở hữu kế hoạch quyết định `ACCEPTED` hoặc `REJECTED`.
- Chỉ kế hoạch `PENDING_PATIENT` được quyết định; quyết định chỉ thực hiện một lần.
- Khi đồng ý, hồ sơ chuyển `IN_TREATMENT`; lưu thời điểm bệnh nhân quyết định.

### 11. Thanh toán

- Chỉ thu tiền cho kế hoạch `ACCEPTED` hoặc `PARTIALLY_PAID`.
- Số tiền dương và không vượt số dư; các phương thức: `CASH`, `CARD`, `TRANSFER`.
- Cập nhật `PARTIALLY_PAID` nếu còn dư, `PAID` nếu đủ; thao tác ghi payment và cập nhật kế hoạch cùng transaction.
- Không xóa giao dịch; hoàn tiền phải là giao dịch/ trạng thái đối ứng để giữ audit.

### 12. Đặt lịch tái khám

- Chỉ tạo sau khi kế hoạch đủ điều kiện theo chính sách (khuyến nghị sau hoàn tất hoặc đã thanh toán).
- Liên kết `parent_plan_id`, giữ nguyên bệnh nhân và bác sĩ điều trị trừ khi được điều phối hợp lệ.
- Vẫn kiểm tra thời gian tương lai, lịch làm việc và xung đột như lịch thường.

### 13. Xem lịch sử điều trị

- Tổng hợp lịch hẹn, hồ sơ khám, chẩn đoán, kế hoạch, dịch vụ và thanh toán.
- Bệnh nhân chỉ xem của mình; bác sĩ xem người mình điều trị; lễ tân chỉ xem dữ liệu cần cho phục vụ; admin xem toàn bộ.
- Lịch sử là dữ liệu đọc, sắp xếp mới nhất trước và có phân trang khi dữ liệu lớn.

## 4. Luồng nghiệp vụ xuyên suốt

```text
Đăng ký/đăng nhập
  -> xem bác sĩ và slot
  -> đặt lịch PENDING
  -> lễ tân CONFIRMED
  -> lễ tân CHECKED_IN
  -> bác sĩ IN_PROGRESS
  -> hồ sơ EXAMINED
  -> kế hoạch PENDING_PATIENT
  -> bệnh nhân ACCEPTED
  -> thanh toán PARTIALLY_PAID/PAID
  -> hoàn tất điều trị
  -> lịch tái khám
  -> lịch sử điều trị
```

## 5. Quy tắc phi chức năng

- Bảo mật: JWT hết hạn, bcrypt, validation server, prepared statement, CORS theo môi trường và không lộ `password_hash`.
- Toàn vẹn: foreign key, unique/check constraint, transaction cho nghiệp vụ nhiều bước.
- Khả dụng: phản hồi lỗi có `error`, `message`, `details`; frontend có loading/error/empty state.
- Audit: lưu `created_at`, `updated_at`, người tạo/người thu/người tiếp nhận.
- Hiệu năng: index lịch theo bác sĩ–thời gian, hồ sơ theo bệnh nhân, payment theo kế hoạch.

## 6. Tiêu chí hoàn thành

- Mỗi yêu cầu có màn hình hoặc API tương ứng và kiểm tra quyền ở backend.
- Test ít nhất: đăng ký trùng email, đặt lịch trùng, chuyển trạng thái sai, bác sĩ sai hồ sơ, bệnh nhân xem nhầm dữ liệu, tính tiền/giảm giá, trả vượt số dư và tái khám trùng slot.
- Frontend gọi đúng endpoint/payload trong `API_CONTRACT.md`; không có dữ liệu demo giả khi chạy chế độ API thật.

