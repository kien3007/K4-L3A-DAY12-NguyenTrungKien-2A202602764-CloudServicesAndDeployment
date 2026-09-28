# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Trung Kiên  Mã học viên: 2A202602764

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên cloud production, giả sử developer quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard hoặc gõ sai tên biến. Nếu để giá trị mặc định `"changeme"`, service vẫn khởi động bình thường, container báo healthy nhưng mở toang endpoint `/ask` ra Internet với một secret phổ biến mà bot quét tự động có thể đoán được, hoặc user lạ có thể gọi API miễn phí và tiêu tốn hàng nghìn USD tiền LLM mà developer không hề hay biết cho đến khi nhận hóa đơn. Ngược lại, khi không có giá trị mặc định, app ném `ValidationError` và crash ngay lúc khởi động (Fail Fast), platform lập tức báo deploy failed ngay trên màn hình deploy, giúp developer phát hiện và bổ sung secret kịp thời trước khi có bất kỳ request nào lọt vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:54:12.345678+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}
```

Hai việc làm được với dòng log JSON này:
1. **Phân tích định lượng và tổng hợp tự động (Metrics & Aggregation):** Các hệ thống thu thập log tập trung (Datadog, AWS CloudWatch, ELK, Loki) có thể parse trực tiếp các trường có kiểu dữ liệu rõ ràng để vẽ biểu đồ và chạy truy vấn như: tính tổng chi phí `sum(cost_usd)` trong ngày, thống kê số lượng token in/out theo từng giờ, hoặc xác định `user_id` nào đang tiêu tốn nhiều ngân sách nhất.
2. **Thiết lập cảnh báo tự động chính xác (Alerting):** Có thể viết rule cảnh báo tự động khi `level == "error"` hoặc khi `cost_usd > 0.05` trên một request duy nhất để bắn thông báo ngay về Slack/PagerDuty của đội ngũ trực vận hành, thay vì phải dùng regex mong manh để tìm kiếm chuỗi văn bản không có ngữ nghĩa như `print("đã trả lời xong")`.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~835 MB) bao gồm:
1. Các công cụ phát triển và biên dịch hệ thống có trong base image đầy đủ `python:3.11` như `gcc`, `g++`, `make`, `build-essential`, các thư viện C headers/dev files, và package manager tools không cần thiết cho môi trường runtime.
2. File tạm và cache trong quá trình cài đặt packages của pip (`.cache`, wheel build intermediates, `.tar.gz`). Ở mô hình Multi-stage build, toàn bộ trình biên dịch và file rác bị cô lập ở stage `builder` rồi bị loại bỏ; stage `runtime` chỉ sử dụng base image siêu nhẹ `python:3.11-slim` và chỉ copy các artifact thư viện đã cài đặt sẵn từ `/install` sang `/usr/local`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer phía trước gồm Base image, `WORKDIR`, `COPY requirements.txt .`, và `RUN pip install ...` hoàn toàn được dùng lại từ Docker cache (CACHED). Docker chỉ bắt đầu chạy lại (invalidate cache) từ layer `COPY app ./app` trở đi. Do đó thời gian build lại chỉ mất 1-2 giây vì không phải tải hay cài lại dependency.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa bất kỳ ký tự nào trong source code, checksum của layer `COPY . .` thay đổi làm toàn bộ cache từ layer đó về sau bị vô hiệu hóa. Lệnh `RUN pip install` sẽ bị ép chạy lại từ đầu mỗi lần sửa code, khiến developer mất từ 1 đến 5 phút cho mỗi lần build image.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python tồn tại một lỗ hổng thực thi mã từ xa (RCE) thông qua input không được sanitize hoặc thư viện bên thứ ba.
2. Kẻ tấn công khai thác RCE để mở một reverse shell vào bên trong container.
3. Vì container không khai báo lệnh `USER`, process chạy với quyền `root` (UID 0) bên trong container.
4. Từ quyền `root` trong container, kẻ tấn công khai thác các lỗ hổng kernel capabilities, mount socket `/var/run/docker.sock`, hoặc lỗ hổng container runtime (như runc breakout) để thoát khỏi ranh giới container (container escape). Vì UID 0 trong container map với UID 0 trên Linux host kernel, kẻ tấn công chiếm luôn quyền root tối cao trên máy host vật lý.
- Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tấn công ngay tại bước 3: Process chỉ chạy dưới quyền của một người dùng thông thường không có đặc quyền (unprivileged user). Kẻ tấn công không thể đọc ghi các file hệ thống, không thể truy cập socket nhạy cảm, và các kỹ thuật container breakout yêu cầu root capability đều bị chặn đứng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa: 20 request trong 2 giây liên tiếp.
- Giải thích:
  Với fixed window (đếm theo phút đồng hồ reset lúc giây :00), người dùng gửi 10 request vào giây cuối cùng của phút thứ nhất (10:00:59). Đến đúng 10:01:00, bộ đếm chuyển sang phút mới và tự động reset về 0. Người dùng lập tức gửi tiếp 10 request vào giây đầu tiên của phút mới (10:01:01). Cả 2 đợt đều hợp lệ theo hạn mức 10 req/phút của từng phút riêng rẽ, nhưng trong khoảng thời gian 2 giây (10:00:59 -> 10:01:01), hệ thống phải chịu tải tới 20 request (gấp đôi sức chịu đựng). Thuật toán sliding window 60s giải quyết triệt để kẽ hở này vì luôn tính tổng số request trong cửa sổ trượt đúng 60 giây lùi về trước tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt:
  - Rate limiting quản lý **tốc độ / tần suất số lượng request** trong một đơn vị thời gian ngắn (ví dụ: 10 request / 60 giây) nhằm bảo vệ hạ tầng máy chủ khỏi nghẽn mạng và tấn công từ chối dịch vụ (DoS).
  - Cost guard quản lý **tổng chi phí tài chính tích lũy** trong một chu kỳ dài (ví dụ: ngân sách tối đa 10 USD / tháng) nhằm bảo vệ ví tiền của bạn khỏi các prompt tiêu tốn quá nhiều token.
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  User chỉ gửi duy nhất 1 request trong ngày (tốc độ 1 req/phút, hoàn toàn nằm trong hạn mức rate limit), nhưng request đó đính kèm văn bản khổng lồ tiêu tốn 100.000 tokens tương đương chi phí $2.0, trong khi ngân sách còn lại của user trong tháng chỉ là $0.5. Cost guard sẽ chặn ngay với mã HTTP 402 Payment Required.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  Vào ngày đầu tháng, user còn nguyên $10.0 ngân sách. User viết script spam 15 câu hỏi ngắn liên tiếp trong 10 giây (mỗi câu chỉ tốn $0.0001, tổng 15 câu chỉ hết $0.0015, ngân sách còn rất nhiều). Tuy nhiên, đến câu thứ 11, Rate limiter sẽ chặn với mã HTTP 429 Too Many Requests vì vi phạm tần suất 10 req/phút.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện sụp đổ dây chuyền (Cascading Failure):
1. Redis gặp sự cố mạng hoặc khởi động lại, mất kết nối trong 30 giây.
2. Endpoint kiểm tra liveness probe gọi vào Redis và bị fail, trả về mã lỗi hoặc timeout.
3. Orchestrator (Docker Swarm/Kubernetes/Cloud Platform) kiểm tra liveness và kết luận rằng toàn bộ 3 process container agent đều đã "chết", từ đó kích hoạt hành động tiêu diệt (kill) và restart lại toàn bộ 3 container agent cùng lúc.
4. Khi Redis vừa hồi phục sau 30 giây, cả 3 container agent vẫn đang bận kẹt trong chu kỳ khởi động lại (booting / crashloop), khiến toàn bộ dịch vụ bị sập hoàn toàn (Total Outage), không còn một container nào phục vụ user.
Sự phân tách đúng: `/health` (liveness) chỉ kiểm tra process app có sống không (không chạm Redis) để orchestrator không restart bừa bãi; `/ready` (readiness) kiểm tra Redis để Load Balancer tạm thời cô lập traffic không gửi vào cho đến khi Redis sẵn sàng trở lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Nếu lưu trong dict Python (in-memory state):
  Với 3 instance agent chạy sau Load Balancer, các request tiếp theo của cùng một user sẽ được phân phối luân phiên (round-robin) đến container A, B hoặc C. Mỗi container chỉ ghi nhớ các tin nhắn gửi trực tiếp tới nó. Do đó, `history_length` sẽ nhảy lộn xộn và không nhất quán (ví dụ: request 1 vào A -> length = 0; request 2 vào B -> length = 0; request 3 vào A -> length = 2; request 4 vào C -> length = 0). Agent bị "mất trí nhớ ngẫu nhiên" giữa các lượt hội thoại.
- Khi lưu trong Redis (Stateless):
  Vì state đã được đưa ra ngoài process vào cụm Redis dùng chung (`history:{user_id}`), dù Load Balancer điều hướng request đến bất kỳ container nào trong 3 container, instance đó đều truy vấn được cùng một lịch sử hội thoại đầy đủ, và `history_length` luôn tăng đều đặn (0 -> 2 -> 4 -> 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:**
  Sau khi build xong trên platform (Railway/Render), service rơi vào trạng thái `Deployment Timed Out` hoặc `Health check failed on port 8000`, container liên tục bị kill và restart.
- **Cách tìm ra nguyên nhân:**
  Kiểm tra logs của container trên Dashboard, phát hiện server Uvicorn in ra log: `Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)`, trong khi các nền tảng PaaS cấp phát một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=7341`) và gửi HTTP health check probe vào cổng `$PORT` đó. Do app hardcode lắng nghe ở cổng 8000 nên probe của cloud không thể kết nối tới app.
- **Cách sửa:**
  Sửa lệnh chạy server trong `Dockerfile` từ cổng cố định sang đọc biến môi trường `$PORT` với giá trị fallback:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`. Nhờ vậy, trên cloud app sẽ lắng nghe đúng cổng do platform gán, còn khi chạy ở máy local app vẫn mặc định chạy cổng 8000.
