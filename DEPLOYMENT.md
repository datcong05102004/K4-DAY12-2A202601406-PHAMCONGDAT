# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phạm Công Đạt |
| Mã học viên | 2A202601406 |
| Repo | https://github.com/datcong05102004/K4-DAY12-2A202601406-PhamCongDat |

## Service

| Mục | Nội dung |
|-----|----------|
| Base URL kiểm tra | `https://chat-production-6ab9.up.railway.app` |
| Platform | Railway |
| Ngày kiểm tra | 2026-08-10 |
| Topology | Railway public domain → service `chat` → Railway Redis |

Service được build từ `Dockerfile` multi-stage và deploy thật lên Railway. Redis là
database service riêng trong cùng project; ứng dụng kết nối qua biến tham chiếu nội bộ
`REDIS_URL` của Railway.

## Biến Môi Trường

Chỉ liệt kê tên và nguồn; không lưu giá trị secret trong repo.

| Biến | Đã set | Nguồn |
|------|--------|-------|
| `PORT` | ✅ | Railway tự gán khi chạy container |
| `API_TOKEN` | ✅ | Railway service variables |
| `REDIS_URL` | ✅ | Tham chiếu `${{Redis.REDIS_URL}}` trong Railway |
| `BUCKET_CAPACITY` | ✅ | Railway service variables |
| `REFILL_PER_MINUTE` | ✅ | Railway service variables |
| `DAILY_BUDGET_USD` | ✅ | Railway service variables |
| `LOG_LEVEL` | ✅ | Railway service variables |
| `LOCAL_FALLBACK` | ✅ | `false` trong `.env` cục bộ để chạy bộ test cloud |

Giá trị secret không được ghi vào tài liệu hoặc commit vào Git. `DEPLOY_API_TOKEN`
chỉ nằm trong `.env` cục bộ để test request có xác thực và có cùng giá trị với
`API_TOKEN` trên Railway.

## Kết Quả Chạy Thật

Trạng thái Railway:

```text
chat     SUCCESS — health check /healthz đạt
Redis    SUCCESS — instance đang chạy
domain   https://chat-production-6ab9.up.railway.app
```

Kết quả endpoint qua HTTPS public domain:

```text
GET  /healthz                 200
GET  /readyz                  200
POST /chat không token        401
POST /chat có Bearer token    200
```

Kết quả rate limit đã được kiểm tra ở stack Docker local:

```text
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

Kiểm tra stateless qua ba replica Docker local với cùng `X-Client-Id`:

```text
turns_before: 0 → 2 → 4 → 6 → 8
```

Log cho thấy cả `chat-1`, `chat-2`, `chat-3` đều nhận request nhưng lịch sử vẫn
liên tục vì được lưu trong Redis.

## Lệnh Kiểm Tra

```powershell
Invoke-RestMethod https://chat-production-6ab9.up.railway.app/healthz
Invoke-RestMethod https://chat-production-6ab9.up.railway.app/readyz
python -m pytest tests/test_cp5.py -v
```

## Ảnh Chụp Màn Hình

Minh chứng stack Docker trước khi deploy được lưu tại
[`screenshots/local-fallback.png`](screenshots/local-fallback.png). Minh chứng cloud
gồm [`screenshots/railway-healthz.png`](screenshots/railway-healthz.png) và
[`screenshots/railway-readyz.png`](screenshots/railway-readyz.png); các ảnh xác nhận
public URL hoạt động và Redis đã sẵn sàng, không chứa giá trị `API_TOKEN`.
