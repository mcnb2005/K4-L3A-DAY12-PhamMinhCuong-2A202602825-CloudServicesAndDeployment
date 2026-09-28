# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định `changeme`, tôi có thể quên cấu hình secret trên Railway mà
> service vẫn báo deploy thành công. Bot quét Internet sau đó có thể gọi
> `/ask` bằng khóa mặc định và làm phát sinh chi phí. Khi khóa không có giá trị
> mặc định, `Settings` báo `ValidationError` ngay lúc container khởi động, nên
> tôi phát hiện cấu hình thiếu trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật tôi nhận được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:02:40.444528+00:00", "user_id": "rate-limit-check", "tokens_in": 302, "tokens_out": 43, "cost_usd": 7.11e-05}`.
> Với JSON này tôi có thể lọc và cộng `cost_usd` theo `user_id`, đồng thời dựng
> cảnh báo hoặc biểu đồ số request theo `level` và `timestamp`. Một câu `print`
> tự do không có các trường ổn định để máy truy vấn như vậy.

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
| 1 stage (bản đầu) | Chưa đo xong: quá trình pull base image đầy đủ bị gián đoạn |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage đo bằng `docker images` là 271 MB và đã dưới giới hạn 500 MB.
> Lần build bản một stage chưa hoàn tất nên tôi chưa ghi một con số không được
> quan sát thật; tôi cần chạy lại lệnh đo trước khi nộp chính thức.
> Stage cuối chỉ mang Python slim, source và các package đã cài. Phần chênh lệch
> so với bản một stage chủ yếu là hệ điều hành Debian đầy đủ, công cụ build,
> cache và các file trung gian chỉ cần trong quá trình cài dependency.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, tạo user, copy
> `requirements.txt` và `pip install` vẫn dùng cache; layer `COPY app ./app` và
> các layer sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi
> thay đổi source sẽ làm mất cache của bước cài dependency, khiến build chậm và
> phải tải/cài lại toàn bộ package dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công trước hết chạy được lệnh
> với quyền của process trong container. Nếu process là root và runtime/container
> còn có lỗ hổng escape hoặc mount nhạy cảm, quyền đó có thể được dùng để sửa file
> hay chiếm quyền cao trên host. `USER appuser` cắt chuỗi ở bước đầu: code bị khai
> thác chỉ có UID 10001 với quyền tối thiểu, không phải root trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request: gửi 10 request ngay trước giây 00, rồi gửi thêm 10 request
> ngay sau khi bộ đếm của phút mới reset. Cả hai nhóm đều hợp lệ theo fixed
> window nhưng lại dồn vào khoảng hai giây; sliding window 60 giây vẫn nhìn thấy
> nhóm trước nên sẽ chặn nhóm sau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
> tổng tiền của từng user trong cả tháng. Một user gửi đều 10 request/phút nhưng
> mỗi request có prompt rất lớn có thể qua rate limit nhưng bị cost guard chặn.
> Ngược lại, một user chưa tiêu gần hết ngân sách nhưng bắn 11 request rất rẻ
> trong vài giây sẽ qua cost guard và bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm cả ba container trả 503 ở endpoint gộp. Orchestrator hiểu
> đó là lỗi liveness nên lần lượt hoặc đồng thời restart cả ba container. Trong
> lúc Redis chưa trở lại, các container mới vẫn fail check và tiếp tục bị restart;
> load balancer không còn instance ổn định để phục vụ. Tách `/health` và `/ready`
> giúp process vẫn sống, còn load balancer chỉ tạm ngừng gửi traffic tới instance
> chưa sẵn sàng.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi thử hai request cùng `X-User-Id` với Redis, tôi quan sát
> `history_length` tăng từ 0 lên 2 vì lượt trước gồm một message user và một
> message assistant. Nếu mỗi container dùng dict Python riêng, request rơi vào
> container khác sẽ có thể quay lại 0; khi tình cờ về container cũ nó lại thấy 2,
> nên dãy số tăng không ổn định và mất hẳn sau khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi dùng local fallback vì workspace chưa có tài khoản/credential cloud. Lỗi
> triển khai thật tôi gặp là Docker báo `Bind for 0.0.0.0:8000 failed: port is
> already allocated`. Tôi chạy `docker ps` và thấy một project khác đang map
> cổng 8000. Tôi không dừng project đó mà đổi mapping Compose thành
> `${HOST_PORT:-8000}:8000`, đặt `HOST_PORT=18000`, rồi chạy lại. Sau đó
> `/health` và `/ready` đều trả 200 trên `http://localhost:18000`.
