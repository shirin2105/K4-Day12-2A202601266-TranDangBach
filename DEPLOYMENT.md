# Thông Tin Deploy — Checkpoint 5

> File thông tin triển khai và kiểm thử dịch vụ Chat Service.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Trần Đăng Bách |
| Mã học viên | 2A202601266 |
| Repo | https://github.com/shirin2105/K4-Day12-2A202601266-TranDangBach |


## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-t07h.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của platform / Upstash |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-chat-t07h.onrender.com/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-chat-t07h.onrender.com/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST https://day12-chat-t07h.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-chat-t07h.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-chat-t07h.onrender.com/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo

```

## Kết Quả Chạy Thật

```text
HTTP/1.1 200 OK
content-type: application/json
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

HTTP/1.1 200 OK
content-type: application/json
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

HTTP/1.1 200 OK
content-type: application/json
{"reply":"Deploy là quá trình đóng gói và đưa ứng dụng lên server công khai...","client_id":"sv-test","turns_before":0,"usd_cost":0.00015,"usage":{"prompt":12,"completion":45}}

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Ảnh đã được đặt trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform Render
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ cURL/trình duyệt

---

## Nếu Dùng Phương Án Dự Phòng

Phương án dự phòng được kích hoạt qua `LOCAL_FALLBACK=true`:
- Đã chạy stack bằng `docker compose up -d` và `fakeredis` trong RAM trên môi trường local.
- Đã kiểm tra đầy đủ liveness `/healthz`, readiness `/readyz`, và xác thực `/chat`.
