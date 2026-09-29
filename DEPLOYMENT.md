# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Van Thanh Huy |
| Mã học viên | 2A202602763 |
| Repo | https://github.com/thanhhuyvan/K4-L3B-DAY12-VanThanhHuy-2A202602763-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3b-day12-vanthanhhuy-2a202602763.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Chỉ ghi tên biến và nguồn giá trị; giá trị bí mật không được lưu trong repo.

| Biến | Đã set | Nguồn / ghi chú |
|------|--------|-----------------|
| `PORT` | ✅ | Render tự gán |
| `AGENT_API_KEY` | ✅ | Render Environment, secret cá nhân |
| `REDIS_URL` | ✅ | Internal connection string của Render Key Value |
| `RATE_LIMIT_PER_MINUTE` | ✅ | Cấu hình Render Environment: 10 |
| `MONTHLY_BUDGET_USD` | ✅ | Cấu hình Render Environment: 10.0 |
| `LOG_LEVEL` | ✅ | Cấu hình Render Environment: INFO |

## Lệnh Kiểm Tra

```bash
# Liveness
curl -i https://k4-l3b-day12-vanthanhhuy-2a202602763.onrender.com/health

# Readiness / Redis
curl -i https://k4-l3b-day12-vanthanhhuy-2a202602763.onrender.com/ready

# Không có API key
curl -i -X POST \
  https://k4-l3b-day12-vanthanhhuy-2a202602763.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# Có API key; khóa được đọc từ môi trường local, không lưu trong repo
curl -i -X POST \
  https://k4-l3b-day12-vanthanhhuy-2a202602763.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: cp5-doc" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

Kiểm tra ngày 2026-09-29:

```text
GET  /health          -> 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready           -> 200 {"status":"ready","redis":true}
POST /ask (không key) -> 401 {"detail":"invalid or missing API key"}
POST /ask (có key)    -> 200, user_id="cp5-doc", có nội dung answer

Rate limit, 15 request liên tiếp với cùng một user:
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — Render hiển thị Deploy succeeded và Live.
- `screenshots/health.png` — URL HTTPS `/health` trả status `ok`.
