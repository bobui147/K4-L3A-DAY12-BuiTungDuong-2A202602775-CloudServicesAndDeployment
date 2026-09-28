# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Các câu trả lời bên dưới dựa trên kết quả kiểm thử local và trạng thái Google Cloud được ghi lại.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Tùng Dương  Mã học viên: 2A202602775

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu tôi quên đặt `AGENT_API_KEY` khi đưa image lên cloud, `Settings` báo lỗi ngay lúc khởi động. Tôi có thể sửa biến môi trường trước khi mở API cho người khác dùng. Nếu code tự nhận khóa mặc định `"changeme"`, service vẫn chạy; người biết khóa mẫu có thể gọi `/ask` và làm phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Tôi gọi `/ask` qua Nginx và lấy được dòng log thật từ container:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:49:13.678550+00:00", "user_id": "scale-verify-20260928", "tokens_in": 1, "tokens_out": 40, "cost_usd": 2.415e-05}`
> Nhờ các trường có cấu trúc, tôi có thể lọc lỗi theo `level` và tính tổng `cost_usd` theo `user_id`. Dòng `print("đã trả lời xong")` không chứa các dữ liệu đó.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f Dockerfile.single -t agent:single .
docker build -t day12-agent:prod .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | khoảng 1730 MB (Docker hiển thị 1.73 GB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo bằng `docker images` sau khi build hai bản với cùng `requirements.txt`: bản multi-stage nhỏ hơn khoảng 1459 MB. Bản ban đầu dùng `python:3.11` đầy đủ, giữ cả môi trường cài package và cache pip trong image. Bản mới dùng `python:3.11-slim`, cài package ở stage builder với `--no-cache-dir` rồi chỉ copy kết quả cần chạy sang stage runtime. Chênh lệch này đến từ cả base image gọn hơn lẫn cách cài dependency; không thể quy hết cho riêng multi-stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi build lại sau khi sửa `app/main.py`, Docker báo các lớp `COPY requirements.txt` và `RUN pip install` là `CACHED`; lớp `COPY app ./app` chạy lại. Nếu `COPY . .` đứng trước `RUN pip install`, thay đổi ở code sẽ làm mất cache của lớp cài thư viện và build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng cho phép chạy lệnh trong app có thể cho kẻ tấn công chiếm quyền trong container. Nếu process là root, việc ghi file hệ thống trong container hoặc khai thác thêm lỗi ở runtime sẽ nguy hiểm hơn cho host. `USER appuser` làm process chỉ có quyền của user thường, hạn chế bước leo thang đầu tiên; nó không thay thế việc vá lỗ hổng hay cô lập container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong khoảng 2 giây: gửi 10 request lúc `10:00:59`, rồi 10 request lúc `10:01:01`. Bộ đếm theo phút đặt lại ở `10:01:00`; cửa sổ trượt 60 giây vẫn nhìn thấy cả hai đợt nên sẽ chặn đợt thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm số lần gọi trong 60 giây; cost guard theo dõi tổng USD trong tháng. Một request có đầu vào rất dài vẫn nằm trong hạn mức số lần nhưng có thể bị chặn do vượt ngân sách. Ngược lại, người dùng gửi 11 request rất rẻ trong một phút có thể chưa hết ngân sách tháng nhưng request thứ 11 bị trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint gộp trả 503 cho cả ba container → bộ điều phối tưởng cả ba process hỏng và restart chúng → các request đang xử lý có thể bị ngắt. Khi Redis trở lại, cụm còn phải khởi động lại trước khi nhận traffic. Tách `/health` để báo process còn sống và `/ready` để báo khả năng dùng Redis giúp load balancer ngừng gửi request trong lúc Redis lỗi mà không restart cả cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy ba agent sau Nginx và gọi `/ask` năm lần với cùng `X-User-Id`. `history_length` lần lượt là `[0, 2, 4, 6, 8]`; log cho thấy request đi vào cả `agent-1`, `agent-2` và `agent-3`. Nếu dùng dict Python, mỗi container có lịch sử riêng, nên con số sẽ nhảy lùi hoặc bắt đầu lại ở 0 khi request sang container khác.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi bật `compute.googleapis.com` cho project `labcloud-510008`, CLI báo `The service is currently being deactivated and deactivation must complete before activation can occur`. Tôi kiểm tra bằng `gcloud services list --available`: Compute và Redis đều `DISABLED`, còn Redis phụ thuộc Compute. Tôi chờ tác vụ vô hiệu hóa ở phía Google Cloud kết thúc rồi chạy lại lệnh bật Compute; lần này thành công. Sau đó tôi bật Redis API, tạo Memorystore và triển khai Cloud Run. `/ready` trên URL công khai trả 200 với `redis:true`, xác nhận đã sửa xong lỗi triển khai.
