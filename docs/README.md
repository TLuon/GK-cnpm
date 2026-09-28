# Tài liệu dự án Hệ thống quản lý phòng khám nha khoa

Đây là bộ đặc tả thống nhất để ba thành viên có thể phát triển song song trong thời gian ngắn mà không làm lệch nghiệp vụ, API hoặc cơ sở dữ liệu.

## Thứ tự nên đọc

1. [Phân tích hệ thống](./SYSTEM_ANALYSIS.md) — tác nhân, đủ 13 yêu cầu, luồng nghiệp vụ và quy tắc trạng thái.
2. [Thiết kế cơ sở dữ liệu](./DATABASE_DESIGN.md) — bảng, khóa, ràng buộc và transaction.
3. [Hợp đồng API](./API_CONTRACT.md) — request/response, mã lỗi và mapping chức năng.
4. [Phân công ba thành viên](./TEAM_TASKS.md) — hai backend/database, một frontend và điểm bàn giao.
5. [Thiết lập môi trường](./SETUP.md) — cách chạy backend, frontend, database và kiểm thử.
6. [Sequence Diagram](./SEQUENCE_DIAGRAM.md) — luồng thời gian của ca sử dụng đặt lịch.
7. [Collaboration Diagram](./COLLABORATION_DIAGRAM.md) — quan hệ đối tượng và thông điệp của ca sử dụng đặt lịch.
8. [Logic và phân quyền](./LOGIC_AND_ROLES.md) — bản tóm tắt tra cứu nhanh.

## Công nghệ thống nhất

- Backend: Node.js, Express, JWT, Zod và `better-sqlite3`.
- Frontend: React, Vite, Fetch API và CSS responsive.
- Database: SQLite cho bài thi/demo; thiết kế có thể chuyển sang PostgreSQL khi triển khai lớn.
- Dữ liệu thời gian truyền qua API theo ISO 8601 UTC; frontend chịu trách nhiệm hiển thị theo múi giờ địa phương.
- Tiền lưu bằng số nguyên VND, tuyệt đối không dùng số thực để cộng tiền.

## Nguyên tắc đồng bộ

- `API_CONTRACT.md` là nguồn sự thật cho endpoint và payload.
- `DATABASE_DESIGN.md` là nguồn sự thật cho tên bảng, cột và enum trạng thái.
- Frontend không tự tính tổng tiền cuối cùng và không tự quyết định quyền.
- Backend không tin `role`, `patientId`, `doctorId` do trình duyệt gửi nếu có thể suy ra từ JWT.
- Mỗi thay đổi hợp đồng phải cập nhật tài liệu trước hoặc cùng commit với code.

