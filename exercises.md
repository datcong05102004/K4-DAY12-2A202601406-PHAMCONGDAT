# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phạm Công Đạt  Mã học viên: 2A202601406

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy nhưng quên đặt `API_TOKEN`. Với cấu hình bắt buộc,
> container dừng ngay và log chỉ thẳng vào biến còn thiếu, nên tôi sửa trước khi
> service nhận traffic. Nếu dùng mặc định `"changeme"`, app vẫn báo healthy và
> người ngoài có thể đoán token đó để gọi `/chat`, làm tiêu quota mà tôi chỉ phát
> hiện sau khi đã phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi thu được là
> `{"event":"chat_completed","severity":"INFO","client_id":"cp4-scale-check","prompt_tokens":43,"completion_tokens":46,"usd_cost":0.00003405}`.
> Từ log có cấu trúc này, tôi có thể lọc chính xác theo `client_id` để điều tra một
> người dùng và cộng `usd_cost`/số token để dựng dashboard hoặc cảnh báo. Câu
> `print("đã trả lời xong")` không chứa các trường ổn định để máy parse và tổng hợp.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo bằng `docker images`: bản một stage là 1.73 GB, bản multi-stage là 270 MB,
> giảm khoảng 84%. Phần chênh lệch chủ yếu đến từ image `python:3.11` đầy đủ và
> các thành phần phục vụ build/cài package. Runtime mới chỉ giữ base slim, thư viện
> đã cài cùng `app/` và `utils/`, không mang toàn bộ build context hay công cụ build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ đổi `app/main.py`, các layer lấy base image, `COPY requirements.txt` và
> `RUN pip install` được dùng lại từ cache. Layer `COPY app/ app/` và các layer sau
> nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source
> đều làm mất cache của layer cài dependency, khiến pip chạy lại dù
> `requirements.txt` không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng thực thi mã từ xa có thể cho kẻ tấn công chạy lệnh với quyền của
> process ứng dụng. Nếu process là root trong container, họ có thể sửa mọi file,
> cài công cụ và tận dụng volume hoặc cấu hình host/Docker socket sai để mở rộng
> ảnh hưởng ra host. `USER appuser` cắt chuỗi ngay sau bước chiếm process: mã độc
> chỉ có UID 10001 với quyền tối thiểu, không mặc nhiên có quyền root trong container.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` là tín hiệu chuẩn để client biết server yêu cầu cơ chế
> Bearer và có thể xử lý 401 đúng cách. Dùng cùng thông báo cho thiếu header, sai
> scheme và sai token tránh tiết lộ bước nào đã đúng; nếu trả lỗi quá chi tiết, kẻ
> dò token có thêm dữ kiện để thu hẹp đầu vào. Chi tiết thật nên nằm trong log nội
> bộ, không nằm trong response công khai.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với giới hạn sức chứa, sau 10 phút xô vẫn chỉ có tối đa 10 token, nên client gửi
> được 10 request liên tiếp rồi request thứ 11 nhận 429. Nếu bỏ `min(capacity, ...)`,
> một xô đã tồn tại và đang rỗng sẽ cộng 10 token/phút × 10 phút = 100 token, cho
> phép burst khoảng 100 request. Nếu đây là client hoàn toàn mới chưa có state thì
> nhánh khởi tạo vẫn trả đúng 10; lỗi tích lũy vô hạn xuất hiện với xô đã tồn tại.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức $30/tháng cho phép sự cố lúc 2 giờ sáng đốt tối đa gần $30 trước khi bị
> chặn và chỉ tự hồi phục khi sang tháng mới. Hạn mức $1/ngày giới hạn thiệt hại
> trong ngày đó ở khoảng $1 cộng tối đa chi phí request cuối theo cơ chế soft
> quota; sang 00:00 UTC key ngày mới được dùng nên service tự phục hồi. Giới hạn
> ngày vì thế thu hẹp đáng kể phạm vi thiệt hại và thời gian phải chờ.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint liveness cũng ping Redis, Redis mất kết nối làm cả ba container trả
> lỗi liveness. Orchestrator kết luận process chết và restart cả ba; chúng khởi
> động lại khi Redis vẫn lỗi, tiếp tục fail rồi rơi vào vòng lặp restart, dù code
> ứng dụng không hỏng. Khi tách riêng, `/healthz` vẫn xác nhận process sống, còn
> `/readyz` trả 503 để load balancer tạm ngừng gửi traffic; container không bị
> restart hàng loạt và tự sẵn sàng lại khi Redis phục hồi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Phiên này tôi dùng local fallback nên chưa phát sinh lỗi dashboard cloud. Lỗi
> triển khai container thực tế tôi gặp là `Bind for 0.0.0.0:8080 failed: port is
> already allocated`. Tôi kiểm tra `docker ps -a` và thấy container Nginx cũ đang
> giữ cổng. Tôi sửa bằng cách chỉ publish một cổng public qua Nginx, chuyển các
> replica `chat` và Redis sang `expose` trong mạng Compose; sau đó ba replica đều
> healthy và gateway trả `/healthz` 200. Khi deploy Railway, cách kiểm tra tương
> tự là đọc build/runtime log và xác nhận `$PORT`, `REDIS_URL` cùng health check.
