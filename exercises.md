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

Nếu để mặc định `"changeme"`, lúc em deploy lên cloud (như Render hay Railway) mà quên không cấu hình biến `AGENT_API_KEY` trong dashboard, app vẫn khởi động bình thường và báo healthy. Khi đó endpoint `/ask` mở công khai ra ngoài mạng với một khóa mặc định rất dễ đoán. Bot tự động quét trúng hoặc người lạ dùng key này gọi API thì em sẽ bị trừ tiền LLM oan mà không hề biết cho tới khi thấy hóa đơn. Ngược lại, khi không có giá trị mặc định, thiếu biến là app crash và báo lỗi `ValidationError` ngay lúc deploy. Thấy dashboard báo đỏ là em biết ngay mình quên biến môi trường để vào bổ sung, tránh được việc lộ app chạy với key mặc định ra Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:54:12.345678+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00015}
```

Hai việc làm được:
1. **Lọc và tính toán chi phí tự động:** Dùng các công cụ quản lý log (như Datadog, Grafana/Loki, CloudWatch) có thể bóc tách thẳng trường `cost_usd` để tính tổng số tiền đã tiêu trong ngày, hoặc nhóm theo `user_id` xem bạn nào đang dùng nhiều token nhất. Nếu chỉ `print` text thông thường thì máy tính không thể tự động thống kê số liệu được.
2. **Cài đặt cảnh báo tự động:** Có thể viết rule giám sát, ví dụ cứ thấy log có `"level": "error"` hoặc trường `cost_usd > 0.05` là tự động bắn thông báo về Telegram/Slack cho mình biết để xử lý. Dùng lệnh `print` thường thì rất khó viết luật lọc và dễ bị sót hoặc báo nhầm.

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

Dung lượng chênh lệch (hơn 800MB) chủ yếu gồm:
- Bản 1 stage dùng base image `python:3.11` đầy đủ, bị dính theo toàn bộ công cụ biên dịch (compiler gcc, g++, make, các file header C/C++) và cache sinh ra trong quá trình chạy `pip install`.
- Bản Multi-stage tách riêng stage `builder` để cài đặt thư viện, sau đó stage `runtime` chỉ dùng base image `python:3.11-slim` rất gọn nhẹ và chỉ copy thư mục gói kết quả `/install` sang. Toàn bộ compiler và file rác trung gian bị bỏ lại hết nên image nhẹ hơn rất nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Docker sẽ dùng lại cache (CACHED) của tất cả các layer trước đó như base image, `COPY requirements.txt` và `RUN pip install`. Nó chỉ chạy lại từ layer `COPY app ./app` trở đi, nên em build lại chỉ mất tầm 1-2 giây là xong.
- Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần em sửa dù chỉ một dòng code trong `app/`, checksum của cả thư mục thay đổi làm Docker hủy bỏ toàn bộ cache từ bước đó. Kết quả là nó bị ép phải chạy lại lệnh `RUN pip install` và tải/cài lại toàn bộ thư viện từ đầu, khiến mỗi lần build mất thêm vài phút rất lâu.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện:
  1. Ứng dụng Python có lỗ hổng (ví dụ RCE - thực thi mã từ xa).
  2. Kẻ tấn công lợi dụng lỗi để mở một shell điều khiển bên trong container.
  3. Do container không khai báo lệnh `USER`, shell này mặc định chạy với quyền `root` (UID 0) trong container.
  4. Từ quyền root này, kẻ tấn công khai thác tiếp các lỗi như mount nhầm socket `docker.sock` hoặc bug bảo mật của container runtime để thoát khỏi container (container escape) ra máy host. Do UID 0 trong container thường tương ứng với UID 0 trên Linux host, kẻ tấn công chiếm luôn quyền root máy chủ.
- Lệnh `USER appuser` cắt đứt chuỗi ở bước 3: Shell của kẻ tấn công chỉ chạy dưới quyền của một user thông thường không có đặc quyền (UID 10001), không thể can thiệp vào file hệ thống và các thủ thuật leo quyền ra máy host đều bị chặn lại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa: 20 request trong 2 giây liên tiếp.
- Giải thích:
  Với cách đếm theo phút đồng hồ (fixed window), user có thể gửi 10 request vào giây cuối cùng của phút trước (10:00:59). Ngay khi bước sang 10:01:00, hệ thống reset bộ đếm về 0, user bắn tiếp luôn 10 request nữa ở giây 10:01:01. Nếu tính theo từng phút thì cả 2 lượt đều hợp lệ (mỗi phút 10 request), nhưng thực tế trong khoảng thời gian chỉ 2 giây (10:00:59 đến 10:01:01), hệ thống đã phải gánh tới 20 request. Dùng thuật toán sliding window 60s sẽ chặn được vì nó luôn tính tổng request trong đúng 60 giây gần nhất tính từ thời điểm gửi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt:
  - Rate limit kiểm soát **tần suất / số lượng request** trong một khoảng thời gian ngắn (ví dụ: 10 request/phút) để tránh nghẽn mạng và chống bị spam.
  - Cost guard kiểm soát **tổng số tiền chi tiêu** trong một chu kỳ dài (ví dụ: tối đa 10 USD/tháng) để tránh bị vung tay quá trán tiền gọi LLM.
- Rate limit cho qua nhưng Cost guard chặn:
  User cả ngày chỉ gửi đúng 1 câu hỏi (hoàn toàn không vi phạm hạn mức 10 req/phút), nhưng lại paste vào cả một tài liệu dài tiêu tốn 100.000 token (tương đương $2), trong khi ngân sách tháng của user chỉ còn $0.5 $\rightarrow$ Cost guard sẽ chặn lại và trả về lỗi 402.
- Cost guard cho qua nhưng Rate limit chặn:
  Đầu tháng ngân sách của user còn nguyên $10. User viết script gửi liên tục 15 câu hỏi ngắn trong 5 giây (mỗi câu chỉ tốn $0.0001, tổng tiền hết chưa đến $0.002, ngân sách còn rất nhiều), nhưng đến câu thứ 11 thì bị Rate limiter chặn và trả về lỗi 429 vì gọi quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố hoặc restart, mất kết nối trong 30 giây.
2. Endpoint kiểm tra liveness gọi vào Redis bị lỗi nên trả về status không thành công.
3. Orchestrator (Docker/K8s) thấy liveness fail liền kết luận là cả 3 container agent đều bị chết/treo, nên ra lệnh kill và restart lại toàn bộ 3 container cùng lúc.
4. Khi Redis hồi phục xong sau 30 giây thì cả 3 container agent vẫn đang bận khởi động lại hoặc bị kẹt crashloop, dẫn đến toàn bộ dịch vụ bị sập hoàn toàn (downtime), không còn container nào nhận request của khách.
*(Tách riêng ra thì `/health` chỉ xem tiến trình Python có sống không để tránh restart nhầm; còn `/ready` mới kiểm tra kết nối Redis để Load Balancer tạm ngừng chia khách vào).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Nếu lưu trong dict Python:
  Vì 3 container agent chạy song song sau Load Balancer, các request tiếp theo của mình sẽ bị chia luân phiên vào container A, B hoặc C. Do mỗi container có một vùng nhớ RAM riêng, `history_length` sẽ nhảy lộn xộn (lúc 0, lúc 2, lúc lại về 0 tuỳ vào việc request rơi trúng container nào). Agent sẽ bị "mất trí nhớ ngẫu nhiên" giữa các câu hỏi.
- Khi lưu trong Redis (Stateless):
  Vì toàn bộ lịch sử được cất vào Redis dùng chung (`history:{user_id}`), nên dù Load Balancer có đẩy request vào container nào thì container đó cũng đọc được cùng một đoạn hội thoại, và `history_length` sẽ tăng đều đặn (0, 2, 4, 6...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lúc deploy lên Render, service bị báo `Deployment timed out` hoặc `Health check failed` rồi container liên tục bị restart.
- **Cách tìm nguyên nhân:** Em vào tab Logs của service trên Render để kiểm tra thì thấy Uvicorn đang chạy ở cổng mặc định 8000 (`http://0.0.0.0:8000`), trong khi Render tự cấp một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `PORT=10000`) và probe của Render gọi vào cổng đó nên không kết nối được.
- **Cách sửa:** Trong file `Dockerfile`, em sửa lại lệnh khởi động thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để server tự động đọc cổng từ biến môi trường `$PORT` do cloud cấp phát.
