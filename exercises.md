# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng hướng dẫn bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Xuân Bách  Mã học viên: 2A202601093

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên Cloud, nếu người quản trị quên cài đặt biến môi trường `AGENT_API_KEY` trong Dashboard và dùng giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động bình thường. Kẻ tấn công hoặc bot tự động có thể đoán API key mặc định `"changeme"` và tự do gọi endpoint `/ask`, tiêu tốn sạch ngân sách LLM của bạn. Việc "fail fast" bắt buộc app báo lỗi và ngắt ngay khi thiếu key giúp lập tức phát hiện cấu hình thiếu trên log deployment trước khi ứng dụng nhận bất kỳ request công khai nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON mẫu:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T09:50:00.123456+00:00", "user_id": "sv-test", "cost_usd": 0.0015, "tokens_in": 12, "tokens_out": 45}`

Hai việc làm được:
1. Các hệ thống quản lý log tập trung (Grafana Loki, Datadog, CloudWatch) có thể tự động parse JSON để lập biểu đồ thống kê chi phí (`cost_usd`) và lượng token tiêu thụ theo thời gian thực, đồng thời tự động phát bẫy cảnh báo khi chi phí tăng đột biến.
2. Dễ dàng truy vấn và lọc log theo từng trường cụ thể (vd: lọc các event của `user_id="sv-test"` hoặc log có `level="error"`) mà không lo vỡ dòng khi chạy đa luồng hoặc scale nhiều instance.

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
| 1 stage (bản đầu) | ~1000 MB |
| Multi-stage | ~220 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~780 MB) bao gồm bộ công cụ biên dịch (gcc, g++, make), thư viện C/C++ header, documentation và cache của pip không cần thiết trong môi trường chạy thật. Multi-stage build đã loại bỏ toàn bộ build-tools này và chỉ sao chép các thư viện đã được cài đặt hoàn chỉnh sang runtime image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa `app/main.py`: Các layer trước đó (`FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install`) đều được dùng lại từ Docker cache. Chỉ layer `COPY app ./app` và các layer phía sau phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`: mỗi lần sửa một dòng code, cache của layer `COPY . .` bị vỡ, làm cho `RUN pip install` buộc phải tải và cài đặt lại toàn bộ thư viện từ đầu, tốn rất nhiều thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Lỗ hổng RCE (Remote Code Execution) trong app cho phép kẻ tấn công gửi payload chạy lệnh shell hệ thống bên trong container. Nếu container chạy dưới quyền root, kẻ tấn công có đầy đủ đặc quyền root trong container, từ đó tiếp tục khai thác các lỗ hổng container escape (hoặc mount socket `/var/run/docker.sock`) để truy cập và kiểm soát toàn bộ máy host với quyền root.
Lệnh `USER appuser` chuyển quyền chạy của container sang tài khoản thường (unprivileged user), khiến kẻ tấn công dù thực thi được lệnh cũng không có quyền sửa đổi hệ thống hay khai thác container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request**.
Giải thích: Người dùng gửi 10 request vào 10:00:58 - 10:00:59 (2 giây cuối của phút thứ nhất). Ngay khi đồng hồ chuyển sang 10:01:00, bộ đếm phút cũ được reset về 0, người dùng tiếp tục gửi 10 request vào 10:01:00 - 10:01:01 (2 giây đầu của phút thứ hai). Tổng cộng họ gửi 20 request trong 2 giây liên tiếp mà vẫn không bị đếm theo phút đồng hồ chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt: Rate limit giới hạn *tần suất request* trong khoảng thời gian ngắn (vd: 10 req/phút) để bảo vệ server khỏi bị quá tải. Cost guard giới hạn *tổng ngân sách (USD)* trong thời gian dài (theo tháng) để bảo vệ tài chính.
- Rate limit cho qua, Cost guard chặn: User gửi 1 request duy nhất trong phút (đạt rate limit), nhưng câu hỏi dài hàng triệu token khiến chi phí lên tới $15, vượt quá ngân sách tháng $10 -> Cost guard chặn (402).
- Cost guard cho qua, Rate limit chặn: User còn nguyên ngân sách $10 nhưng gửi liên tiếp 20 request ngắn trong 5 giây -> Rate limit chặn ở request thứ 11 (429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối trong 30 giây -> Endpoint gộp trả về lỗi 503 HTTP.
2. Orchestrator (Docker/Kubernetes) thấy Liveness check (`/health`) trả 503 liên tục -> Đánh giá cả 3 container `agent` đã bị hỏng.
3. Orchestrator ra lệnh tiêu diệt và khởi động lại (restart) toàn bộ 3 container.
4. Do Redis vẫn chưa kết nối lại được, các container mới khởi động lên lại bị rớt healthcheck và bị restart lặp đi lặp lại (CrashLoopBackOff), làm tê liệt hoàn toàn hệ thống và ngắt các request đang xử lý dở.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python (in-memory): do Load Balancer phân phối luân phiên request tới 3 instance A, B, C khác nhau, mỗi instance sẽ giữ bộ nhớ RAM riêng. Bạn sẽ thấy `history_length` tăng giảm bất thường (ví dụ: request 1 vào A được 0, request 2 vào B lại là 0, request 3 vào A được 2, request 4 vào C lại là 0). Ngược lại, khi lưu ở Redis, cả 3 instance cùng đọc/ghi chung một nguồn nên `history_length` tăng đều đặn (0, 2, 4, 6,...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Healthcheck timeout / Port binding failure trên Railway deployment.
Thông báo lỗi: `Application failed to respond on PORT 6421`.
Nguyên nhân: Code cũ hardcode uvicorn bind vào cổng `8000`, trong khi các nền tảng PaaS như Railway tự động gán một cổng ngẫu nhiên thông qua biến môi trường `$PORT`.
Cách sửa: Sửa `app/config.py` đọc trường `port` từ biến `PORT`, và điều chỉnh `Dockerfile` / command chạy uvicorn đọc động `${PORT:-8000}`.
