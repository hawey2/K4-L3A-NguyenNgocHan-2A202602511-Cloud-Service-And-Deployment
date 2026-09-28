# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn *` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Ngoc Han  Mã học viên: 2A202602511

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu app có giá trị mặc định `"changeme"` cho `AGENT_API_KEY`, khi deploy lên cloud mà quên set biến môi trường này, app vẫn khởi động thành công và lắng nghe request. Kẻ tấn công có thể biết trước khóa mặc định này (vì nó nằm trong code công khai) và gọi API miễn phí, tiêu tốn ngân sách LLM của bạn trong khi bạn nghĩ rằng app đã được bảo vệ. Với fail fast (không có default), app sẽ crash ngay lúc khởi động với `ValidationError`, buộc bạn phải set `AGENT_API_KEY` trước khi deploy được — điều này ngăn chặn việc lộ ngân sách hoàn toàn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")` không làm được.

> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:15:30.123456+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.000123}

Hai việc làm được với JSON log mà `print()` không làm được:
1. **Lọc và truy vấn tự động**: Có thể dùng log aggregation (như Datadog, Loki, Elasticsearch) để query `user_id="sv01"` và tính tổng `cost_usd` trong ngày, hoặc đếm số request bị rate limit (level="warning") mỗi phút — không thể làm được với plain text log.
2. **Cảnh báo tự động**: Thiết lập alert khi `cost_usd` của một user vượt ngưỡng hoặc tỷ lệ error (level="error") tăng đột biến trong 5 phút — máy đọc JSON và trigger alert, còn `print()` chỉ để người đọc mắt.

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
| 1 stage (bản đầu) | ~1.1 GB |
| Multi-stage | ~180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~920 MB) chủ yếu là: (1) **Compiler và build tools** (gcc, make, python3-dev) trong stage builder dùng để compile các package có C extension (như `redis`, `uvloop`), bị loại bỏ ở stage runtime vì chỉ copy `/install` (site-packages) sang; (2) **Cache pip** (`--no-cache-dir` giúp giảm nhưng stage đơn vẫn giữ cache layer); (3) **Base image đầy đủ** `python:3.11` (~1GB) thay vì `python:3.11-slim` (~120MB); (4) **Source code và file không cần thiết** (`.git`, `__pycache__`, tests) bị copy vào image ở bản single-stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile multi-stage hiện tại:
> - **Cache hit**: `FROM python:3.11-slim AS builder`, `WORKDIR /app`, `COPY requirements.txt`, `RUN pip install --prefix=/install -r requirements.txt` — các layer này không thay đổi.
> - **Cache miss (chạy lại)**: `FROM python:3.11-slim AS runtime`, `COPY --from=builder /install /usr/local`, `COPY app ./app`, `COPY utils ./utils`, `RUN useradd...`, `USER appuser`, `HEALTHCHECK`, `CMD` — vì `COPY app ./app` thay đổi do sửa file `main.py`.
>
> Nếu đặt `COPY . .` trước `RUN pip install`: Mỗi lần sửa 1 ký tự trong code, Docker invalidate cache từ layer `COPY . .` trở đi, buộc **cài lại toàn bộ dependency** (`pip install`) — mất vài phút mỗi lần build thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) Code Python có lỗ hổng RCE (ví dụ `eval(user_input)` hoặc path traversal); (2) Kẻ tấn công gửi payload khai thác, chạy được lệnh shell trong container; (3) Vì container chạy root, shell có UID 0 — quyền cao nhất trong container; (4) Nếu container mount volume host hoặc có cấu hình `--privileged` / `--cap-add=SYS_ADMIN` / docker socket mount, kẻ tấn công **breakout** ra host với quyền root.  
> Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi tại **bước (3)**: shell khai thác chỉ có quyền user thường, không thể ghi file hệ thống, không thể mount, không thể escape container dễ dàng — giảm thiểu tác động của RCE.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **20 request** trong 2 giây.
>
> Cách đạt: User gửi 10 request lúc **10:00:59.500** (vẫn trong phút 10:00), rồi ngay lập tức gửi 10 request nữa lúc **10:01:00.500** (vừa sang phút 10:01, bộ đếm reset). Cả 20 request xảy ra trong khoảng 1 giây, nhưng đều "hợp lệ" theo logic đếm phút đồng hồ vì chúng rơi vào 2 cửa sổ phút khác nhau. Sliding window không có kẽ hở này vì cửa sổ luôn trượt 60 giây về phía sau request hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau**: Rate limit giới hạn **số lượng request** (tần suất), cost guard giới hạn **chi phí tiền USD** (tổng token tiêu tốn).
>
> - **Rate limit cho qua, cost guard chặn**: User gửi 5 request/phút (dưới rate limit 10/phút), nhưng mỗi request là prompt 50.000 token → chi phí ~$0.05/request. Sau 5 request đã tốn $0.25, vượt ngân sách $0.10/tháng → cost guard chặn (402).
> - **Cost guard cho qua, rate limit chặn**: User gửi 15 request/phút, mỗi request chỉ 100 token (rẻ, ~$0.0001). Tổng chi phí $0.0015 < ngân sách $10 → cost guard cho qua. Nhưng rate limit 10/phút → request thứ 11 bị chặn (429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện khi gộp endpoint và kiểm tra Redis:
> 1. Redis mất kết nối (network partition / restart).
> 2. `/health` (gộp) của **3 container** đều gọi `Redis.ping()` → thất bại → trả 503.
> 3. Orchestrator (K8s / Docker Swarm / Railway) nhận 503 từ liveness probe → coi container **unhealthy**.
> 4. Orchestrator **restart 3 container cùng lúc** (vì cả 3 đều unhealthy).
> 5. Container mới khởi động, tiếp tục gọi `/health` → Redis vẫn chưa xong → 503 → restart loop.
> 6. Hệ thống **đứng hoàn toàn** (0 container phục vụ) trong 30 giây, dù Redis sẽ tự hồi phục.
> 7. Khi Redis hồi phục, container mới start xong mới phục hồi traffic — downtime kéo dài thêm vài phút do restart storm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (stateless): `history_length` **tăng dần đều** (0, 2, 4, 6...) bất kể request rơi vào container nào, vì 3 container chia sẻ cùng 1 Redis.
>
> Với dict Python trong RAM (stateful): `history_length` **nhảy loạn / reset ngẫu nhiên**. Ví dụ: request 1 vào container A → `history_length=0` → lưu vào dict của A. Request 2 vào container B → `history_length=0` (B không biết A đã lưu gì) → agent "mất trí nhớ". Request 3 quay lại A → `history_length=2`. Con số không tăng đều, phụ thuộc vào load balancer phân phối request vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi: **Health check timeout** trên Railway sau khi deploy thành công. Log: `Health check failed after 30 attempts`.
>
> Nguyên nhân: App bind cổng cứng `8000` trong code (`uvicorn --port 8000`), nhưng Railway gán cổng ngẫu nhiên qua biến môi trường `$PORT`. Container lắng nghe trên 8000, Railway health check gọi vào `$PORT` (ví dụ 32768) → connection refused.
>
> Cách tìm ra: Xem log Railway dashboard → thấy `uvicorn running on http://0.0.0.0:8000` nhưng health check URL là `https://app.up.railway.app` (port 443, map đến `$PORT` nội bộ). So sánh với local Docker Compose dùng `PORT=8000` vẫn chạy được.
>
> Sửa: Đổi `CMD` trong Dockerfile thành `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]` để đọc `$PORT` từ môi trường, fallback 8000 cho local. Build lại, deploy → health check pass.