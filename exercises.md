# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời thực tế.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Van Thanh Huy  Mã học viên: 2A202602763

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc deploy service công khai nhưng tôi quên khai
> báo `AGENT_API_KEY` trên cloud. Nếu khóa có mặc định là `"changeme"`, app vẫn
> báo khỏe và người ngoài có thể đoán khóa này để gọi `/ask`, làm phát sinh chi
> phí. Khi trường này bắt buộc, service báo lỗi cấu hình ngay khi endpoint cần
> nạp settings; tôi phát hiện việc thiếu biến trước khi coi deployment là hoàn
> tất và thêm secret trong Render Environment.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log tôi thu được sau khi gọi `/ask` trên container local:
>
> ```json
> {"event":"ask_completed","level":"info","timestamp":"2026-09-29T05:22:32.718618+00:00","user_id":"exercise-log","tokens_in":3,"tokens_out":41,"cost_usd":2.505e-05}
> ```
>
> Từ log JSON này tôi có thể (1) lọc hoặc đếm các event `ask_completed` theo
> `user_id` và khoảng thời gian, và (2) cộng `tokens_in`, `tokens_out` hoặc
> `cost_usd` để làm dashboard/cảnh báo chi phí. Chuỗi `print("đã trả lời xong")`
> không có field cố định để máy truy vấn và cũng không cho biết user, token hay
> chi phí.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile một-stage từ lịch sử Git thành `agent:single` và
> đo được 1.73 GB; image hiện tại `agent:multi` là 271 MB, giảm khoảng 1.46 GB
> (xấp xỉ 84%). Bản đầu dùng base `python:3.11` đầy đủ và giữ toàn bộ package,
> công cụ hệ thống cùng layer cài đặt trong image chạy. Bản mới dùng
> `python:3.11-slim` cho runtime và chỉ copy thư viện Python từ builder, nên
> không mang các thành phần chỉ phục vụ build sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer `FROM`, `WORKDIR`, `COPY
> requirements.txt` và `RUN pip install` được dùng lại từ cache vì dependency
> không đổi. Layer `COPY app ./app` phải tạo lại; các layer đứng sau nó cũng
> được đánh giá lại, nhưng chúng nhẹ vì không cài package. Nếu đặt `COPY . .`
> trước `RUN pip install`, bất kỳ thay đổi source nào cũng làm layer `COPY` đổi
> và buộc `pip install` chạy lại, khiến mỗi vòng build chậm hơn nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh từ xa, kẻ tấn công trước
> hết sẽ chạy lệnh với UID của process trong container. Nếu process là root,
> họ có toàn quyền trong container; kết hợp với cấu hình nguy hiểm như mount
> Docker socket, mount thư mục host có quyền ghi hoặc một lỗ hổng kernel, họ có
> thể sửa/chạy workload khác và tiến tới quyền cao trên host. `USER appuser`
> cắt chuỗi ở bước đầu: mã bị chiếm chỉ có UID 10001 và quyền file hạn chế.
> Đây là giảm thiểu thiệt hại, không thay thế việc vá lỗ hổng hay cấu hình
> container an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây: gửi 10 request
> ở cuối phút, ví dụ `10:00:59.x`, rồi gửi thêm 10 request ngay sau khi bộ đếm
> reset ở `10:01:00.x`. Cả hai phút lịch đều không vượt 10, nhưng thực tế có 20
> request dồn sát nhau. Sliding window nhìn lại đúng 60 giây nên chặn đợt thứ
> hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ tốc độ trong cửa sổ ngắn (10 request/60 giây cho mỗi
> user), còn cost guard bảo vệ tổng chi phí tích lũy trong tháng. Một user gửi
> đều 1 request mỗi phút sẽ luôn qua rate limit nhưng cuối cùng có thể vượt
> ngân sách tháng và bị cost guard chặn. Ngược lại, khi ngân sách còn nhiều,
> user gửi 11 request rất rẻ trong vài giây thì cost guard vẫn cho phép theo
> tiền, nhưng rate limit chặn request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp hai endpoint, Redis mất kết nối sẽ làm probe chung trả lỗi dù process
> FastAPI vẫn sống. Load balancer lần lượt loại cả ba replica khỏi danh sách
> nhận traffic; sau số lần probe thất bại quy định, orchestrator còn có thể
> restart cả ba. Container mới vẫn kiểm tra Redis đang lỗi nên lại bị đánh dấu
> unhealthy, tạo vòng restart và service mất hoàn toàn khả dụng. Tách endpoint
> giúp `/health` tiếp tục báo process còn sống, còn `/ready` chỉ tạm ngừng đưa
> traffic vào replica cho tới khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với ba replica cùng dùng Redis, `history_length` của cùng một `X-User-Id`
> vẫn theo một chuỗi chung, ví dụ `0, 2, 4, 6...` vì mỗi request ghi hai message
> và request sau đọc được dữ liệu do bất kỳ replica nào ghi. Nếu dùng một dict
> Python, mỗi replica chỉ có lịch sử riêng: kết quả có thể nhảy như `0, 0, 2,
> 0, 2...` tùy request được cân bằng vào container nào; restart container còn
> làm phần lịch sử của nó trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế của tôi là Render báo deployment `Live` và `/health` trả 200,
> nhưng `/ready` và `/ask` đều trả `500 Internal Server Error`. Tôi gọi riêng
> ba endpoint để khoanh vùng rồi kiểm tra `Settings`: `AGENT_API_KEY` là trường
> bắt buộc, còn `REDIS_URL` mặc định trỏ localhost. Nguyên nhân là web service
> được tạo nhưng chưa có đủ environment variables và chưa nối Render Key
> Value. Tôi tạo `day12-redis`, lấy internal connection string, đặt
> `REDIS_URL` và `AGENT_API_KEY` trong Render Environment rồi deploy lại. Sau
> đó `/ready` trả 200, `/ask` thiếu key trả đúng 401 và `/ask` có key trả 200.
