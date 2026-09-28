# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Trung Kiên |
| Mã học viên | 2A202602764 |
| Repo | https://github.com/kien3007/K4-L3A-DAY12-NguyenTrungKien-2A202602764-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-ycr6.onrender.com |
| Platform | Render (Web Service + Managed Redis Instance) |
| Region | Singapore (ap-southeast-1) |
| Ngày deploy | 28/09/2026 |
| Trạng thái | Live / Healthy 🟢 |

---

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của platform (day12-redis connectionString) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-ycr6.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-ycr6.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-ycr6.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-ycr6.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-ycr6.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

---

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 09:20:30 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 5bcf4ff8-7b89-447f
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a421ac2daa9391ad-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-agent","version":"1.0.0"}
```

# 2. Readiness
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 09:20:51 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: d3504177-f4c6-45a6
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a421acb2dbc0aedc-SIN
alt-svc: h3=":443"; ma=86400

{"status":"ready","redis":true}
```

# 3. Không có API key
HTTP/1.1 401 Unauthorized
Date: Mon, 28 Sep 2026 09:22:52 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 635b0668-2568-41fb
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a421afa5ad3af910-SIN
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing API key"}
```

# 4. Có API key
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 09:52:11 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
rndr-id: aec43766-b19b-4607
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-cache-status: DYNAMIC
CF-RAY: a421da933cd3ce46-SIN
alt-svc: h3=":443"; ma=86400

{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên. (Mình đang nhớ 20 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":20,"cost_usd":9.57e-05,"tokens":{"in":450,"out":47}}(.venv)
```

```text
# 5. Rate limit (15 lần gọi liên tiếp)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
