# Hướng dẫn thiết lập môi trường

## 1. Yêu cầu máy

- Node.js 20 LTS trở lên.
- npm 10 trở lên.
- Git.
- Không cần cài SQLite server; `better-sqlite3` tạo file database tự động.

Kiểm tra:

```powershell
node --version
npm --version
git --version
```

## 2. Clone dự án

```powershell
git clone https://github.com/TLuon/GK-cnpm.git
Set-Location GK-cnpm
```

## 3. Cấu hình backend

Tạo `.env` ở thư mục gốc từ mẫu:

```env
PORT=3000
DATABASE_PATH=./data/clinic.db
JWT_SECRET=thay-bang-chuoi-ngau-nhien-dai
CORS_ORIGIN=http://localhost:5173
```

Quy tắc:

- Không commit `.env` hoặc database thật.
- Production bắt buộc thay `JWT_SECRET` mạnh.
- `CORS_ORIGIN` là danh sách origin được phép, không dùng `*` khi có cookie/thông tin nhạy cảm.

Cài và chạy:

```powershell
npm install
npm run dev
```

Kiểm tra health:

```powershell
Invoke-RestMethod http://localhost:3000/api/health
```

## 4. Cấu hình frontend

Trong terminal khác:

```powershell
Set-Location frontend
npm install
```

Tạo `frontend/.env`:

```env
VITE_API_URL=http://localhost:3000/api
VITE_DEMO_MODE=false
```

Chạy:

```powershell
npm run dev
```

Mở `http://localhost:5173`.

## 5. Database và seed

- Khi backend chạy lần đầu, tạo `data/clinic.db`, bảng, index và dữ liệu seed.
- Migration phải có version; không xóa database để “sửa” schema trong môi trường có dữ liệu.
- Tài khoản seed chỉ dùng local/demo và phải đổi hoặc tắt ở production.

Tài khoản demo đề xuất:

| Role | Email | Mật khẩu |
|---|---|---|
| ADMIN | `admin@clinic.local` | `Admin@123` |
| RECEPTIONIST | `reception@clinic.local` | `Reception@123` |
| DOCTOR | `doctor@clinic.local` | `Doctor@123` |

Bệnh nhân nên đăng ký qua UI để kiểm tra yêu cầu số 1.

## 6. Chạy kiểm thử

Backend:

```powershell
npm test
```

Frontend production build:

```powershell
Set-Location frontend
npm run build
```

Các test bắt buộc trước demo:

1. Đăng ký và đăng nhập.
2. Đặt lịch thành công và đặt trùng trả `409`.
3. Lễ tân xác nhận/check-in; role khác bị `403`.
4. Bác sĩ được giao mới tạo hồ sơ.
5. Tính đúng giá/giảm giá.
6. Bệnh nhân khác không duyệt plan.
7. Thanh toán một phần, đủ tiền và chặn vượt số dư.
8. Tái khám chặn slot trùng.
9. Lịch sử chỉ đúng phạm vi quyền.

## 7. Thứ tự chạy demo

```text
Backend :3000 -> Frontend :5173 -> Đăng ký bệnh nhân
-> Đặt lịch -> Lễ tân tiếp nhận -> Bác sĩ khám/lập kế hoạch
-> Bệnh nhân xác nhận -> Thanh toán -> Tái khám -> Lịch sử
```

## 8. Lỗi thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| `401 INVALID_TOKEN` | Token hết hạn/sai secret | Đăng nhập lại, kiểm tra cùng `JWT_SECRET` |
| CORS trên trình duyệt | Origin frontend chưa khai báo | Sửa `CORS_ORIGIN=http://localhost:5173` |
| `409 SLOT_UNAVAILABLE` | Slot vừa được người khác đặt | Tải lại availability và chọn slot mới |
| Frontend báo network error | Sai `VITE_API_URL` hoặc API chưa chạy | Kiểm tra health và `.env`, khởi động lại Vite |
| SQLite locked | Transaction giữ quá lâu/nhiều tiến trình | Giữ transaction ngắn, bật WAL/busy timeout |
| UI vẫn dùng demo | `VITE_DEMO_MODE=true` | Đổi `false`, khởi động lại Vite |

## 9. Checklist trước khi nộp

- `.env`, `data/*.db`, `node_modules`, `frontend/dist` nằm trong `.gitignore`.
- Không có mật khẩu/secret thật trong Git.
- `npm test` và frontend build đều pass.
- Mermaid trong bốn tài liệu liên quan hiển thị được trên GitHub.
- Demo đủ chuỗi quyền PATIENT → RECEPTIONIST → DOCTOR → PATIENT/RECEPTIONIST.

