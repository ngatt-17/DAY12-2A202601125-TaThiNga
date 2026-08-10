# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tạ Thị Nga  Mã học viên: 2A202601125

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

 > Nếu deploy mà quên đặt `AGENT_API_KEY`, app dừng ngay khi khởi động thay vì chạy công khai với khóa `changeme`. Nhờ vậy mình phát hiện lỗi cấu hình trước khi service nhận request và tránh để người khác dùng một API key mặc định để tiêu ngân sách.


### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

 > Ví dụ: `{"event":"ask_completed","level":"info","user_id":"sv-test","tokens_in":4,"tokens_out":36,"cost_usd":2.22e-05}`. Từ log JSON, hệ thống có thể lọc theo `user_id` để điều tra một request và tính tổng `cost_usd` hoặc số token. Các công cụ log cũng có thể parse từng trường để cảnh báo khi chi phí tăng; một dòng `print` chỉ là văn bản tự do, khó tìm kiếm và tổng hợp chính xác.


### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

 > Mình chưa có số đo bản 1-stage trong lần build này; bản multi-stage được build từ `python:3.11-slim` và chỉ copy dependency cùng source sang runtime. Chênh lệch của bản 1-stage thường đến từ compiler, file tạm và cache của quá trình cài package bị giữ lại trong image cuối. Multi-stage loại các phần build đó khỏi runtime nên image nhỏ hơn và ít công cụ thừa hơn.


### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

 > Khi chỉ sửa một ký tự trong `app/main.py`, các layer từ `FROM` đến `RUN pip install` vẫn được dùng lại vì `requirements.txt` không đổi. Layer `COPY --chown=appuser:appuser . .` phải chạy lại và image runtime được tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mỗi lần sửa source sẽ làm layer source thay đổi trước, khiến Docker phải chạy lại `pip install` dù dependency không đổi.


### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

 > Một lỗ hổng cho phép kẻ tấn công thực thi lệnh trong process Python. Nếu container chạy bằng root, process đó có quyền đọc hoặc sửa nhiều tài nguyên trong container và có thể lợi dụng lỗ hổng container để tiếp cận host với quyền cao. `USER appuser` làm process ứng dụng bắt đầu với quyền thường, nên dù ứng dụng bị khai thác thì quyền trực tiếp của kẻ tấn công cũng bị giới hạn.


### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

 > Với cách đếm theo phút đồng hồ, người dùng có thể gửi tối đa 20 request trong khoảng 2 giây: 10 request ngay trước thời điểm chuyển phút, rồi thêm 10 request ngay sau khi bộ đếm reset ở giây 00. Sliding window 60 giây kiểm tra timestamp từng request nên không cho phép khoảng dồn request này.


### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

 > Rate limit giới hạn số lần gọi trong một cửa sổ thời gian, còn cost guard giới hạn tổng tiền mỗi user trong tháng. Một request có prompt hoặc output rất lớn có thể vẫn nằm dưới giới hạn số request nhưng bị cost guard chặn vì vượt ngân sách. Ngược lại, nhiều request nhỏ có thể chưa tốn hết ngân sách nhưng vẫn bị rate limit chặn vì gửi quá nhanh.


### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

 > Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả 3 container sẽ trả health fail. Load balancer coi cả cụm là không khỏe và ngừng gửi traffic, còn orchestrator có thể lần lượt restart các container. Khi Redis hoạt động lại, các container mới phải khởi động và kiểm tra lại dependency, gây gián đoạn không cần thiết. Vì vậy `/health` chỉ kiểm tra process, còn `/ready` mới kiểm tra Redis.


### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

 > Với Redis, các container cùng đọc một lịch sử nên request đầu tiên có `history_length` bằng 0, request tiếp theo thấy 2 message trước đó, dù request đi vào instance nào. Nếu dùng một dict Python, mỗi container có bộ nhớ riêng: request chuyển sang instance mới có thể lại thấy `history_length` bằng 0 hoặc một giá trị khác. Scale ngang khi đó làm lịch sử hội thoại không nhất quán.


### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

 > Lỗi mình gặp trên Railway là Uvicorn báo `Invalid value for '--port': '$PORT' is not a valid integer`. Mình xem deployment log và thấy `railway.toml` truyền literal `$PORT`, vì Railway không shell-expand chuỗi `startCommand` đó. Mình sửa lệnh thành `sh -c 'exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'`, deploy lại và kiểm tra `/health` trả 200, `/ready` trả 200 với Redis.
