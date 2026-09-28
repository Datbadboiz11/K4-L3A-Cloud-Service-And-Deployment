# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Thân Tiến Đạt  Mã học viên: 2A202603023

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> **Tình huống:** Khi triển khai ứng dụng lên môi trường Cloud/Staging, người vận hành sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
> - Nếu để giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động bình thường và báo 200 OK. Khi đó các bot quét mạng hoặc kẻ xấu có thể dễ dàng đoán được key mặc định phổ biến này để gọi API `/ask` miễn phí, đốt sạch toàn bộ ngân sách LLM mà ta không hề hay biết cho đến khi nhận hóa đơn.
> - Khi không có mặc định (Fail-fast), `pydantic-settings` sẽ ném lỗi `ValidationError` và tiến trình lập tức dừng ngay lúc khởi động. Alert trên hệ thống giám sát báo đỏ ngay, buộc người deploy phải cấu hình khóa bí mật hợp lệ trước khi hệ thống có thể nhận request thực tế.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:15:30.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.00012}`
>
> **Hai việc làm được với dòng log JSON này:**
> 1. **Truy vấn và thống kê định lượng theo trường (Structured Aggregation):** Các hệ thống quản lý log tập trung (như Datadog, Grafana Loki, CloudWatch) có thể tự động bóc tách các trường `cost_usd`, `tokens_in`, `tokens_out` để vẽ dashboard trực quan về tổng chi phí AI theo thời gian thực và xếp hạng người dùng tiêu thụ nhiều tài nguyên nhất. `print()` thuần túy không thể làm được điều này nếu không viết regex phức tạp.
> 2. **Thiết lập cảnh báo tự động (Automated Alerting):** Có thể thiết lập luật cảnh báo tự động gửi về Slack/PagerDuty khi trường `level == "error"` hoặc phát hiện các request có `cost_usd` vượt ngưỡng bất thường.

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
| 1 stage (bản đầu) | ~1050 MB |
| Multi-stage | 272 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~780 MB) bao gồm:
> - Base image `python:3.11` đầy đủ chứa Debian base lớn, toàn bộ bộ công cụ biên dịch (`build-essential`, `gcc`, `make`, `g++`), các file header C (`.h`), thư viện tĩnh (`.a`), documentation, man pages và các tiện ích hệ điều hành dành cho quá trình compile/build.
> - Trong bản Multi-stage, stage `builder` dùng để cài đặt thư viện vào thư mục `/install`, sau đó stage `runtime` sử dụng `python:3.11-slim` chỉ copy đúng các package đã cài đặt sang `/usr/local`. Toàn bộ cache của pip, các công cụ build nặng nề đều bị loại bỏ hoàn toàn, giúp image nhỏ gọn, khởi động nhanh và giảm bề mặt tấn công bảo mật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Với Dockerfile hiện tại:** Các layer cài đặt phụ thuộc gồm `COPY requirements.txt .` và `RUN pip install ...` ở phía trên không thay đổi nên Docker tái sử dụng lại 100% từ Cache (`CACHED`). Docker chỉ phải chạy lại từ layer `COPY . .` trở đi (thao tác này hoàn tất chỉ trong vài phần trăm giây).
> - **Nếu đặt `COPY . .` lên trước `RUN pip install`:** Khi ta thay đổi dù chỉ một ký tự trong `app/main.py`, hash của layer `COPY . .` sẽ bị thay đổi. Theo nguyên lý Docker Layer Cache, toàn bộ các layer phía dưới nó đều bị vô hiệu hóa (cache invalidated). Khi đó, Docker bắt buộc phải tải và cài đặt lại toàn bộ thư viện từ `requirements.txt` từ đầu, khiến thời gian build tăng từ vài giây lên vài phút mỗi lần cập nhật code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> **Chuỗi sự kiện tấn công leo thang đặc quyền:**
> 1. Kẻ tấn công phát hiện và khai thác một lỗ hổng trong code ứng dụng (ví dụ: lỗi Remote Code Execution - RCE qua deserialization hoặc command injection).
> 2. Kẻ tấn công thực thi được shell bên trong container. Do container mặc định chạy quyền `root`, process của hacker có UID 0 bên trong container.
> 3. Lợi dụng quyền root này, hacker có thể khai thác các lỗ hổng nhân Linux (kernel exploit) hoặc truy cập các volume gắn kết nhạy cảm (như docker socket `/var/run/docker.sock`) để thoát khỏi container (container breakout).
> 4. Vì UID 0 trong container mặc định ánh xạ với UID 0 (root) của máy host, kẻ tấn công chiếm toàn quyền kiểm soát máy chủ vật lý bên ngoài.
>
> **Lệnh `USER` cắt đứt chuỗi ở bước 2:** Lệnh `USER appuser` buộc tiến trình Python chạy với người dùng thông thường không có đặc quyền (UID 10001). Khi bị khai thác RCE, kẻ tấn công chỉ có quyền hạn cực kỳ hạn chế bên trong thư mục app, không có quyền can thiệp vào kernel, không thể gắn kết thiết bị hay thực thi các lệnh hệ thống để leo thang đặc quyền ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Người dùng có thể gửi tối đa: **20 request** trong vòng 2 giây liên tiếp.
> - **Cách đạt được con số đó:**
>   + Người dùng gửi 10 request vào giây `10:00:59` (giây cuối cùng của phút thứ 00). Hệ thống kiểm tra thấy trong phút 00 mới có 10 request nên cho qua toàn bộ.
>   + Đúng 1 giây sau, đồng hồ chuyển sang `10:01:00`, bộ đếm theo phút tự động reset về 0. Người dùng gửi tiếp 10 request nữa ngay tại giây này. Bộ đếm tính cho phút 01 mới có 10 request nên tiếp tục cho qua.
>   + Như vậy chỉ trong 2 giây từ `10:00:59` đến `10:01:00`, server phải chịu tải 20 request (gấp đôi hạn mức).
> - Thuật toán **Sliding Window** (cửa sổ trượt) giải quyết được vấn đề này vì nó luôn tính tổng số request trong khoảng thời gian động `[now - 60s, now]`, đảm bảo tại bất kỳ cửa sổ 60 giây liên tục nào cũng không vượt quá 10 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Sự khác biệt:**
> - **Rate Limiter:** Giới hạn theo **số lượng request trong một khoảng thời gian ngắn** (ví dụ: 10 request/phút) để bảo vệ server khỏi bị nghẽn mạng và quá tải tài nguyên xử lý (CPU/RAM).
> - **Cost Guard:** Giới hạn theo **tổng chi phí tài chính thực tế tích lũy** (ví dụ: $10.0/tháng) nhằm bảo vệ ngân sách chi trả cho nhà cung cấp mô hình LLM.
>
> **Tình huống Rate Limit cho qua nhưng Cost Guard chặn:**
> - Người dùng chỉ gửi 1 request trong 1 phút (hoàn toàn đúng luật rate limit), nhưng câu hỏi đó đính kèm tài liệu dài tới 100,000 token, chi phí xử lý vượt quá ngân sách còn lại trong tháng của người dùng ➔ Cost Guard chặn ngay trước khi gọi LLM với mã `402 Payment Required`.
>
> **Tình huống ngược lại (Cost Guard cho qua nhưng Rate Limit chặn):**
> - Người dùng mới sử dụng đầu tháng, ngân sách còn nguyên $10.0, nhưng gửi dồn dập 20 câu hỏi ngắn (mỗi câu chỉ 5 token, chi phí chỉ $0.0001) trong vòng 3 giây ➔ Tổng chi phí rất nhỏ chưa chạm trần ngân sách, nhưng Rate Limiter lập tức chặn từ request thứ 11 với mã `429 Too Many Requests` để tránh làm tê liệt hệ thống.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> **Thứ tự sự kiện (Hiệu ứng thác đổ - Cascading Failure):**
> 1. **Giây 0:** Redis gặp trục trặc mạng hoặc khởi động lại, tạm thời không phản hồi trong 30 giây.
> 2. **Giây 5:** Bộ điều phối Cloud (Docker Compose / Kubernetes) thăm dò Liveness probe vào `/health`. Do `/health` kiểm tra kết nối Redis và thất bại, probe trả về lỗi.
> 3. **Giây 10 - 15:** Liveness probe thất bại liên tiếp đạt ngưỡng restart. Bộ điều phối nhận định tiến trình container đã bị treo/hỏng và đồng loạt ra lệnh khởi động lại (restart) cả 3 container agent.
> 4. **Giây 15 - 30:** Cả 3 container khởi động lại nhưng Redis vẫn chưa phục hồi, liveness probe tiếp tục thất bại, khiến cả cụm rơi vào vòng lặp restart liên tục (CrashLoopBackOff).
> 5. **Giây 30:** Khi Redis đã hoạt động trở lại, các container vẫn đang bận restart và chưa kịp khởi tạo xong, dẫn đến toàn bộ hệ thống bị sập hoàn toàn (downtime kéo dài) thay vì chỉ cần tạm dừng tiếp nhận request mới trong 30 giây đó.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - **Khi lưu bằng Redis (Stateless):** `history_length` tăng đều đặn và nhất quán qua mỗi lượt hỏi: 0 ➔ 2 ➔ 4 ➔ 6 ➔ 8... bất kể request được Load Balancer điều phối vào container nào trong 3 container, vì toàn bộ instance đều đọc và ghi chung vào Redis.
> - **Nếu lưu trong dict Python (Stateful trong RAM):** Do Load Balancer phân phối tải luân phiên (Round-Robin), request 1 đến container A (A lưu 2 tin), request 2 đến container B (B chưa có tin nào nên báo `history_length = 0`), request 3 đến container C (C cũng báo `history_length = 0`), request 4 lại quay về container A (A báo `history_length = 2`). Con số `history_length` sẽ nhảy lộn xộn, agent bị "mất trí nhớ" và trả lời sai lệch vì ngữ cảnh hội thoại bị phân mảnh giữa các container.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Lỗi gặp phải:** Liveness probe thất bại dẫn đến service bị deploy fail trên Cloud (`Healthcheck failed on /health: Connection refused`).
> - **Cách tìm nguyên nhân:** Mở tab Deploy Logs trên Dashboard của Railway/Render. Nhận thấy ứng dụng in log `Uvicorn running on http://0.0.0.0:8000`, trong khi nền tảng Cloud tự động gán một cổng ngẫu nhiên qua biến môi trường `$PORT` (ví dụ `PORT=6543`) và kiểm tra sức khỏe ở cổng đó.
> - **Cách sửa:** Cập nhật lại lệnh CMD trong `Dockerfile` và `railway.toml` thành `uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}`. Khi đó app tự động ưu tiên đọc biến `$PORT` do platform cấp thay vì cố định cổng 8000. Sau khi cấu hình lại, container vượt qua health check ngay lập tức.
