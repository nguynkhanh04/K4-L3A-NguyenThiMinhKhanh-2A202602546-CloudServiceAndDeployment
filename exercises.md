# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Cau tra loi cua ban*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Thi Minh Khanh  Mã học viên: 2A202602546

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định "changeme", khi deploy lên cloud mà quên cấu hình biến môi trường, ứng dụng vẫn chạy và ai cũng có thể dùng api key "changeme" để dùng chùa hệ thống (gọi LLM), làm phát sinh chi phí lớn trước khi tôi phát hiện ra. Việc "chết sớm" giúp tôi biết ngay lúc deploy là thiếu cấu hình quan trọng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T14:24:00+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.0001}`
Hai việc làm được: (1) Truy vấn lọc các log theo user_id để tính tổng chi tiêu hoặc số token của một user cụ thể; (2) Tự động đẩy log vào công cụ giám sát (như Datadog/ELK) để tạo dashboard cảnh báo nếu chi phí vọt lên cao bất thường.

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
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch là những build tools (như gcc, make), OS packages phục vụ cho việc biên dịch, pip cache và các file rác phát sinh trong quá trình cài đặt thư viện. Trong multi-stage, các thứ này bị bỏ lại ở stage builder, chỉ những kết quả cần thiết mới được mang sang runtime stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Layer cài đặt OS dependencies và pip install (từ COPY requirements.txt) được lấy từ cache. Chỉ các layer từ `COPY . .` trở về sau mới phải chạy lại.
Nếu đặt `COPY . .` trước `RUN pip install`, thì việc sửa 1 ký tự trong code cũng làm layer `COPY . .` mất cache, dẫn đến việc phải chạy lại `RUN pip install` tốn rất nhiều thời gian mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu có lỗ hổng như Remote Code Execution trong app Python, hacker sẽ thực thi lệnh với quyền root (quyền của process app). Nếu cấu hình runtime không chặt (ví dụ không namespace), root trong container có thể map đến root trên host, giúp hacker kiểm soát host. 
Lệnh `USER` chuyển process sang chạy bằng user thường ngay từ đầu, nên hacker chỉ chiếm được quyền user thường, không thể phá hoại sâu vào container hoặc host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa có thể gửi 20 request trong 2 giây. Ví dụ: gửi 10 request lúc 10:00:59 (thuộc phút 00) và 10 request lúc 10:01:00 (thuộc phút 01). Việc này không vi phạm hạn mức 10/phút nhưng lại gây tải đột ngột 20 req/2s. Cửa sổ trượt khắc phục được điều này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit đếm số lượng request, còn cost guard đếm ngân sách tiền điện toán.
- Rate cho qua nhưng cost chặn: User gọi rất ít request (vd 2 req/phút) nhưng câu hỏi dài cả cuốn sách, tốn 10$ tiền token, vượt budget tháng.
- Cost cho qua nhưng rate chặn: User liên tục gọi 50 req/phút bằng những câu hỏi cực ngắn "hi", tốn rất ít tiền, nhưng chạm rate limit.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

(1) Redis mất kết nối. (2) Health check của cả 3 container đều thất bại do không ping được Redis. (3) Orchestrator (Docker/K8s) cho rằng cả 3 container đều bị lỗi process nên ra lệnh SIGKILL và khởi động lại toàn bộ. (4) Khi Redis hồi phục, 3 container vẫn đang trong quá trình boot, dẫn đến hệ thống bị sập diện rộng (cascading failure). Nếu tách riêng, /ready fail chỉ làm LB ngừng đẩy traffic, tránh việc container bị kill oan.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu dùng dict Python, các request từ 1 user có thể được load balancer chia cho 3 container khác nhau. Vì mỗi container giữ một dict riêng (nhớ được những câu nó xử lý), `history_length` sẽ trồi sụt ngẫu nhiên thay vì tăng dần một cách đồng nhất, làm agent như bị "mất trí nhớ" từng khúc.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi gặp phải: Không có tài khoản cloud để thực hành deploy.
Cách giải quyết: Đã bật `LOCAL_FALLBACK=true` trong `.env` để sử dụng phương án dự phòng (chạy docker compose ở máy). 

---
