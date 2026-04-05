# Hướng Dẫn Kéo Dự Án GoClaw Về Máy Mới (Backup & Restore)

Tài liệu này ghi chú lại chi tiết các bước chuyển hệ thống GoClaw và toàn bộ Database, Token từ máy đang phát triển sang một máy cá nhân mới hoàn toàn (hoặc thư mục clone mới).

---

## Bước 1: [MÁY CŨ] Trích Xuất Dữ Liệu & Backup
Để đảm bảo code mang sang máy mới có thể mở và dùng ngay toàn bộ dữ liệu (lịch sử chat, Agents, cấu hình Provider, v.v.), bạn cần thu thập 2 món quan trọng nhất:

1. **Dump file Database Postgres:**
   Chạy lệnh sau ở thư mục gốc project đang hoạt động để bơm DB ra file `.sql`:
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.postgres.yml exec -T postgres pg_dump -U goclaw -d goclaw > .data/database_backup.sql
   ```

2. **Bảo tồn file chìa khóa mã hóa (`.env`):**
   Bạn hãy copy lại nguyên xi file `.env` (phải đảm bảo file này đang chứa 2 thông số cũ là `GOCLAW_GATEWAY_TOKEN` và `GOCLAW_ENCRYPTION_KEY`). Đây là chìa khóa duy nhất để giải mã dữ liệu lấy từ file Data bên trên.

*(Khuyến nghị: Tải cả `.env` và `database_backup.sql` lên Google Drive hoặc Gửi qua Zalo cá nhân, rồi mới gõ git push code lên Github. Tránh push lộ bí mật lên repo chung).*

---

## Bước 2: [MÁY MỚI] Tải Code Và Ráp Cầu Chì
Khi về nhà hoặc sang máy tính mới:

1. **Clone project:**
   ```bash
   git clone git@github.com-lightla:lightla/ai-go-claw.git
   cd ai-go-claw
   ```
2. **Setup Cấu Hình Bắt Buộc:**
   - Tạo file `.env` và dán nguyên nội dung từ bước 1 vào.
   - Tranh thủ ném file `database_backup.sql` vào thư mục `.data/database_backup.sql`.
   
   ⚠️ **LƯU Ý:** Tránh tuyệt đối cái lỗi lặp lại lệnh `./goclaw onboard` vì nó sẽ phá hủy cấu hình cũ của bạn và sinh ra Key mã hóa mới.

3. **Gắn Cổng Giao Diện Web (Nếu bạn đổi Port):**
   Nếu trong file `.env` gốc, bạn đã chỉnh `GOCLAW_PORT` sang một số khác `18790` (ví dụ `18791`) để tránh xung đột trừng lặp, thì bắt buộc bạn phải sang thư mục `ui/web` tạo `.env` cho Frontend:
   ```bash
   cd ui/web 
   # Cập nhật thông số để Frontend chọc đúng đường dẫn gọi Backend:
   VITE_BACKEND_PORT=18791
   VITE_BACKEND_HOST=localhost
   VITE_WS_URL=ws://localhost:18791/ws
   ```
   
4. **Xóa Biến Gây Nhiễu Giữa Docker & Terminal (Tẩy Não Terminal):**
   Nếu bạn đã lỡ vọc Terminal hiện tại, hãy ngắt mọi luồng Token giả chặn lại:
   ```bash
   unset GOCLAW_GATEWAY_TOKEN 
   unset GOCLAW_ENCRYPTION_KEY
   ```

---

## Bước 3: [MÁY MỚI] Phục Hồi Và Đồng Bộ Vận Hành
Quá trình này tuyệt đối phải cô lập và bơm Postgres trước khi chạy Backend:

1. **Dọn dẹp môi trường sạch tinh tươm:**
   Xóa rác để chắc cú tránh vụ lỗi Schema:
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.postgres.yml down -v
   ```

2. **Bơm Data vào Cổng Postgres Nổi (Cấm ép Backend chạy theo):**
   Chạy DUY NHẤT con container Postgres (Chờ ~5s cho Up and Running healthy):
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.postgres.yml up -d postgres
   ```
   Sau đó "nạp linh lực" bằng lệnh này:
   ```bash
   cat .data/database_backup.sql | docker compose -f docker-compose.yml -f docker-compose.postgres.yml exec -T postgres psql -U goclaw -d goclaw
   ```

3. **Cập nhật & Chạy Thả Xích Backend (Đồng bộ Code & DB v29):**
   Thêm cờ `--build` cực kỳ quan trọng để bắt Docker không lấy file ảo cũ trên mạng mà nhào nặn lại Code bằng Image mới nhất ứng với Schema Database mang sang:
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.postgres.yml up -d --build
   ```

---

## Bước 4: Kiểm Tra Giao Diện Cuối
Đạt đủ quy trình đó, giờ chỉ việc thưởng thức thành quả:
- Mở terminal sang thư mục Web:
  ```bash
  cd ui/web
  pnpm install
  pnpm dev
  ```
- Trình duyệt truy cập `http://localhost:5173/` (Hoặc Port định sẵn).
- Ngay chỗ Đăng Nhập, nhập **User ID** là `system`, và **Gateway Token** chính là đoạn mã dài lưu trong `GOCLAW_GATEWAY_TOKEN` của file `.env`. Done! 🎉


```
// ui/web/.env 
VITE_BACKEND_PORT=18791
VITE_BACKEND_HOST=localhost
VITE_WS_URL=ws://localhost:18791/ws

// .env
GOCLAW_GATEWAY_TOKEN=*
GOCLAW_ENCRYPTION_KEY=*

POSTGRES_PASSWORD=
POSTGRES_PORT=5433
GOCLAW_PORT=18791
GOCLAW_UI_PORT=3001
```