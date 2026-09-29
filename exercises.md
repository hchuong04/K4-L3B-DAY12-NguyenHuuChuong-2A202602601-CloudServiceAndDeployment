# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: Thay thế dòng câu trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hữu Chương  Mã học viên: 2A202602601

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để giá trị mặc định là `"changeme"`, khi deploy lên môi trường production mà quên khai báo biến môi trường `AGENT_API_KEY`, ứng dụng vẫn khởi động bình thường. Kẻ tấn công có thể dễ dàng đoán khóa mặc định này để gọi API miễn phí và làm cạn kệt ngân sách LLM. Ngược lại, cơ chế Fail Fast khiến ứng dụng crash ngay lập tức lúc boot, buộc lập trình viên/DevOps phải bổ sung biến môi trường hợp lệ trước khi hệ thống có thể nhận traffic công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"timestamp": "2026-09-29T05:46:19.661640+00:00", "level": "info", "event": "ask_completed", "service": "day12-agent", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 0.00002265}
```

Hai việc làm được:
1. **Truy vấn & Lọc log tự động (Structured Querying):** Các hệ thống tập trung log (như Datadog, ELK, Grafana Loki) có thể parse cấu trúc JSON tự động để lọc log theo `user_id`, tính tổng `cost_usd` hoặc tìm nhanh các log có `level == "error"`.
2. **Giám sát & Cảnh báo thời gian thực (Metrics & Alerting):** Có thể trích xuất các trường định lượng (`cost_usd`, `tokens_out`) để vẽ biểu đồ theo dõi chi phí LLM theo thời gian thực và kích hoạt cảnh báo tự động nếu chi phí tăng đột biến.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 215 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800MB) bao gồm các công cụ biên dịch (gcc, g++, make), thư viện C header, apt package manager cache và bộ công cụ Python build không cần thiết cho môi trường runtime. Dùng `python:3.11-slim` cùng kĩ thuật multi-stage loại bỏ toàn bộ các công cụ build này khỏi image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer cài đặt dependency (`COPY requirements.txt .` và `RUN pip install ...`) được dùng lại từ Docker layer cache. Chỉ có layer `COPY . .` và các bước cấu hình user/CMD sau đó mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần thay đổi code trong `app/main.py`, cache tại bước `COPY . .` sẽ bị vô hiệu hóa (invalidated), khiến Docker phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu, làm tăng đáng kể thời gian build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện:** Giả sử code Python mắc lỗ hổng Remote Code Execution (RCE). Kẻ tấn công gửi payload thực thi lệnh shell tùy ý. Nếu container chạy với quyền `root` (UID 0), tiến trình shell đó mang quyền root trong container. Kẻ tấn công tiếp tục khai thác lỗ hổng container escape (hoặc ghi vào volume/socket được mount từ host) để chiếm quyền điều khiển hệ thống máy host dưới quyền root.
- **Điểm cắt đứt:** Lệnh `USER appuser` ép tiến trình container chạy dưới một người dùng không đặc quyền (non-root, UID 10001). Khi đó, kẻ tấn công dù khai thác được RCE cũng chỉ mang quyền hạn hạn chế của `appuser`, không thể can thiệp vào các tài nguyên hệ thống hoặc leo leo quyền trên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**.
Giải thích: Người dùng gửi 10 request vào giây `10:00:59` (giây cuối của phút 00) và gửi tiếp 10 request vào giây `10:01:01` (giây đầu của phút 01). Vì hệ thống reset bộ đếm lúc giây 00, cả hai đợt request đều hợp lệ theo hạn mức từng phút nhưng làm bùng nổ 20 request chỉ trong 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác biệt:** Rate Limit bảo vệ hệ thống khỏi nổ tải trong ngắn hạn (tần suất call/phút). Cost Guard bảo vệ tài chính khỏi vượt hạn mức chi tiêu trong dài hạn (tổng chi phí USD/tháng).
- **Rate Limit cho qua nhưng Cost Guard chặn:** User gọi 1 request/ngày nhưng prompt quá dài tiêu tốn hết $10.0 ngân sách tháng -> Rate Limit cho qua (1 req/phút < 10 req/phút) nhưng Cost Guard chặn với mã 402 Payment Required.
- **Cost Guard cho qua nhưng Rate Limit chặn:** User mới dùng còn nguyên $10.0 ngân sách nhưng gửi liên tiếp 15 request nhỏ trong vòng 3 giây -> Cost Guard cho qua (chưa hết $10) nhưng Rate Limit chặn từ request thứ 11 với mã 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối 30 giây.
2. Endpoint gộp bị thất bại do không kết nối được Redis.
3. Liveness probe thất bại, orchestrator (Docker/Kubernetes) phán đoán tiến trình app đã bị đơ/chết.
4. Orchestrator lập tức diệt và tái khởi động (restart) toàn bộ 3 container `agent`.
5. Cụm container rơi vào vòng lặp crash/restart, làm sập hoàn toàn dịch vụ ngay cả với các endpoint không dùng tới Redis.
*(Nếu tách riêng: `/health` vẫn trả 200 OK để giữ container sống, chỉ `/ready` trả 503 để Load Balancer tạm thời ngưng đẩy traffic cho đến khi Redis khôi phục)*.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

`history_length` sẽ **thay đổi thất thường và không đồng nhất giữa các request**. Vì Load Balancer phân phối các request theo cơ chế Round-Robin sang 3 container `agent` độc lập, mỗi container giữ một dict Python riêng trong bộ nhớ RAM. Do đó, request 1 rơi vào container 1 (len=1), request 2 rơi vào container 2 (len=1), request 3 rơi vào container 1 (len=2)... khiến trải nghiệm người dùng bị sai lệch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** Khi gọi `curl https://agent-production-39f4.up.railway.app/health`, Railway trả về: `HTTP/1.1 404 Not Found` kèm body `{"status":"error","code":404,"message":"Application not found"}`.
- **Cách tìm nguyên nhân:** Chạy lệnh `railway domain status`, phát hiện domain công khai của Railway chưa được liên kết với `Target port` của container (mặc định Target Port là `-`), trong khi ứng dụng uvicorn bên trong container chạy ở cổng dynamic 8080 do Railway cấp.
- **Cách sửa:** Chạy lệnh `railway domain update agent-production-39f4.up.railway.app --port 8080` để định tuyến lưu lượng từ domain của Railway vào cổng 8080 của container.
