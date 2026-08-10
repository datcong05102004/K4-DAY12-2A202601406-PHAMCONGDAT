# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phạm Công Đạt |
| Mã học viên | 2A202601406 |
| Repo | https://github.com/datcong05102004/K4-DAY12-2A202601406-PHAMCONGDAT |

## Service

| Mục | Nội dung |
|-----|----------|
| Base URL kiểm tra | `http://localhost:8000` |
| Platform | Local fallback bằng Docker Compose; nền tảng cloud mục tiêu là Railway |
| Ngày kiểm tra | 2026-08-10 |
| Topology | Nginx → 3 replica chat → Redis |

Phiên làm bài này sử dụng phương án local fallback vì chưa thực hiện bước đăng nhập
và xác minh tài khoản Railway/Render. Stack được build từ cùng Dockerfile production,
chạy ba replica ứng dụng qua Nginx và dùng chung Redis.

## Biến Môi Trường

Chỉ liệt kê tên và nguồn; không lưu giá trị secret trong repo.

| Biến | Đã set | Nguồn |
|------|--------|-------|
| `PORT` | ✅ | Giá trị mặc định 8000; cloud có thể tự gán |
| `API_TOKEN` | ✅ | File `.env` cục bộ, không được Git theo dõi |
| `REDIS_URL` | ✅ | Docker Compose đặt thành Redis nội bộ |
| `BUCKET_CAPACITY` | ✅ | Cấu hình ứng dụng |
| `REFILL_PER_MINUTE` | ✅ | Cấu hình ứng dụng |
| `DAILY_BUDGET_USD` | ✅ | Cấu hình ứng dụng |
| `LOG_LEVEL` | ✅ | Cấu hình ứng dụng |
| `LOCAL_FALLBACK` | ✅ | Bật cho bài kiểm tra CP5 cục bộ |

Khi deploy thật lên Railway, cần tạo Redis add-on và đặt `API_TOKEN`, `REDIS_URL`
trong dashboard. Giá trị không được ghi vào tài liệu hoặc commit vào Git.

## Kết Quả Chạy Thật

Stack:

```text
chat-1   healthy
chat-2   healthy
chat-3   healthy
redis    healthy
nginx    running tại localhost:8000
```

Kết quả endpoint qua Nginx:

```text
GET  /healthz                 200
GET  /readyz                  200
POST /chat không token        401
POST /chat có Bearer token    200
```

Kết quả gọi 15 lần với cùng một client để kiểm tra rate limit:

```text
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Kiểm tra stateless qua ba replica với cùng `X-Client-Id`:

```text
turns_before: 0 → 2 → 4 → 6 → 8
```

Log cho thấy cả `chat-1`, `chat-2`, `chat-3` đều nhận request nhưng lịch sử vẫn
liên tục vì được lưu trong Redis.

## Lệnh Kiểm Tra

```powershell
docker compose up -d --build --scale chat=3
docker compose ps
Invoke-RestMethod http://localhost:8000/healthz
Invoke-RestMethod http://localhost:8000/readyz
python -m pytest tests/test_cp5.py -v
```

## Ảnh Chụp Màn Hình

Ảnh Docker Desktop hoặc terminal cần được chụp thủ công vào `screenshots/` trước
khi nộp để thể hiện ba replica healthy và kết quả gọi endpoint. Không chụp hoặc
đưa giá trị `API_TOKEN` vào ảnh.
