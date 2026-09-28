# Thông Tin Deploy — Checkpoint 5

> Báo cáo triển khai môi trường Cloud thực tế cho AI Agent Service.
> Toàn bộ dịch vụ được cấu hình theo chuẩn 12-Factor App, chạy trên container bảo mật và kết nối dịch vụ Redis độc lập.
>
> **Lưu ý bảo mật:** Chỉ ghi TÊN biến môi trường và nguồn gán giá trị, tuyệt đối không lưu giá trị secret/API key trong tài liệu này.

---

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Trung Kiên |
| Mã học viên | 2A202602764 |
| Repository | https://github.com/kien3007/K4-L3A-DAY12-NguyenTrungKien-2A202602764-CloudServicesAndDeployment |

---

## Service Trên Cloud

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-ycr6.onrender.com |
| Platform | Render (Web Service + Managed Redis Instance) |
| Region | Singapore (ap-southeast-1) |
| Ngày deploy | 28/09/2026 |
| Trạng thái | Live / Healthy 🟢 |

---

## Biến Môi Trường Đã Set Trên Cloud

Toàn bộ biến cấu hình và secret được quản lý trong mục **Environment** trên Render Dashboard:

| Biến môi trường | Đã set | Nguồn giá trị / Ghi chú |
|-----------------|:------:|-------------------------|
| `PORT` | ✅ | Platform Render tự động cấp phát khi khởi tạo container |
| `AGENT_API_KEY` | ✅ | Tạo bằng `secrets.token_urlsafe(32)`, lưu an toàn trên Render Dashboard |
| `REDIS_URL` | ✅ | Internal Connection String từ Redis add-on (`day12-redis`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | `10` (Giới hạn 10 request/phút theo thuật toán Sliding Window ZSET) |
| `MONTHLY_BUDGET_USD` | ✅ | `10.0` (Ngân sách tối đa $10/tháng cho mỗi user) |
| `LOG_LEVEL` | ✅ | `INFO` (Structured JSON logging ra stdout) |

---

## Lệnh Kiểm Tra

### 1. Trên Linux / macOS / Bash

```bash
# 1. Liveness Probe — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-ycr6.onrender.com/health

# 2. Readiness Probe — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-ycr6.onrender.com/ready

# 3. Không có API Key — mong đợi 401 Unauthorized
curl -i -X POST https://day12-agent-ycr6.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API Key hợp lệ — mong đợi 200 OK kèm câu trả lời
curl -i -X POST https://day12-agent-ycr6.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy"}'

# 5. Kiểm tra Rate Limit — gọi 15 lần liên tiếp (10 lần đầu 200, 5 lần sau 429)
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-ycr6.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

### 2. Trên Windows PowerShell

```powershell
# 1. Liveness Probe
curl.exe -i https://day12-agent-ycr6.onrender.com/health

# 2. Readiness Probe
curl.exe -i https://day12-agent-ycr6.onrender.com/ready

# 3. Không có API Key (Mong đợi 401)
curl.exe -i -X POST https://day12-agent-ycr6.onrender.com/ask -H "Content-Type: application/json" -d '{\"question\":\"Hello\"}'

# 4. Có API Key hợp lệ (Mong đợi 200 OK)
curl.exe -i -X POST https://day12-agent-ycr6.onrender.com/ask -H "Content-Type: application/json" -H "X-API-Key: $env:AGENT_API_KEY" -H "X-User-Id: sv-test" -d '{\"question\":\"Deploy\"}'

# 5. Kiểm tra Rate Limit (15 lần gọi liên tiếp)
1..15 | ForEach-Object { curl.exe -s -o NUL -w "%{http_code} " -X POST https://day12-agent-ycr6.onrender.com/ask -H "Content-Type: application/json" -H "X-API-Key: $env:AGENT_API_KEY" -H "X-User-Id: sv-test" -d '{\"question\":\"test\"}' }; Write-Host ""
```

---

## Kết Quả Chạy Thật

Trích xuất kết quả chạy thực tế từ môi trường máy trạm gọi lên server Render:

```http
# 1. Liveness (/health)
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 09:20:30 GMT
Content-Type: application/json
Connection: keep-alive
Server: cloudflare
x-render-origin-server: uvicorn

{"status":"ok","service":"day12-agent","version":"1.0.0"}
```

```http
# 2. Readiness (/ready)
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 09:20:51 GMT
Content-Type: application/json
Connection: keep-alive
Server: cloudflare
x-render-origin-server: uvicorn

{"status":"ready","redis":true}
```

```http
# 3. Không có API key (POST /ask)
HTTP/1.1 401 Unauthorized
Date: Mon, 28 Sep 2026 09:22:52 GMT
Content-Type: application/json
Connection: keep-alive
Server: cloudflare
x-render-origin-server: uvicorn

{"detail":"invalid or missing API key"}
```

```http
# 4. Có API key hợp lệ (POST /ask)
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 09:24:15 GMT
Content-Type: application/json
Connection: keep-alive
Server: cloudflare
x-render-origin-server: uvicorn

{"answer":"Câu hỏi hay. Deploy thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 6 lượt trao đổi trước đó.)","user_id":"anonymous","history_length":6,"cost_usd":4.695e-05,"tokens":{"in":137,"out":44}}
```

```text
# 5. Rate limit (15 lần gọi liên tiếp)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

---

## Ảnh Chụp Màn Hình Minh Chứng

Các ảnh minh chứng được lưu trữ trong thư mục `screenshots/` theo đúng quy định:

* `screenshots/dashboard.png` — Trang quản lý Web Service và Redis trên Render Dashboard.
* `screenshots/health.png` — Kết quả gọi endpoint `/health` trả về mã 200 kèm JSON trạng thái.
