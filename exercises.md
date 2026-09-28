# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Mỗi câu bên dưới là phần phản ánh dựa trên code, test và các phép đo đã ghi rõ.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hoàng Công Minh  Mã học viên: 2A202602774

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ, khi cấu hình app service trên Railway mà quên đặt `AGENT_API_KEY`,
> mình muốn bước kiểm tra cấu hình báo lỗi ngay, để sửa trước khi mở API.
> Nếu dùng khóa mặc định `changeme`, app vẫn nhận request và người khác có
> thể đoán khóa để gọi `/ask`. Ở code hiện tại, `Settings` chưa được tạo
> ngay trong `lifespan`, nên thiếu key thực tế có thể chỉ lộ ra khi endpoint
> cần cấu hình được gọi; muốn fail fast đúng lúc startup thì cần gọi
> `get_settings()` trong `lifespan`. Đây là điểm mình cần cải thiện, không
> phải một lỗi deploy mình đã quan sát thấy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Trong test gọi `/ask` với key hợp lệ, mình thu được dòng log (test dùng
> `user_id=sv-test`, không chứa câu hỏi hay API key):
>
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:11:43.934157+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`
>
> Với các trường có cấu trúc, mình có thể (1) lọc/đếm số lần `ask_completed`
> theo `user_id` và khoảng thời gian, và (2) cộng `cost_usd` theo user để
> theo dõi chi phí/cảnh báo. Dòng `print("đã trả lời xong")` không có dữ liệu
> để làm hai việc đó.

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
| 1 stage (bản đầu, `python:3.11`) | 1.73 GB |
| Multi-stage (`python:3.11-slim`) | 271 MB |
| 1 stage đối chứng (cùng base `python:3.11-slim`) | 320 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Mình build cả Dockerfile một-stage ban đầu lẫn bản multi-stage và đọc cột
> `DISK USAGE` bằng `docker image ls`: **1.73 GB** so với **271 MB**. Chênh
> lệch lớn này có hai nguyên nhân: bản đầu dùng base `python:3.11` đầy đủ
> thay vì bản slim, và nó giữ cả nội dung `COPY . .` cùng cache/phần phục vụ
> cài đặt trong image chạy. Để tách riêng ảnh hưởng của multi-stage, mình
> build thêm bản một-stage **cùng base slim**: 320 MB, vẫn lớn hơn bản
> multi-stage **49 MB**. Runtime multi-stage chỉ nhận dependency đã cài và
> hai thư mục `app/`, `utils/`, không mang nguyên stage builder theo.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm tạm một comment vào `app/main.py` rồi build lại. Output Docker
> cho thấy `COPY requirements.txt`, `RUN pip install` ở builder và
> `COPY --from=builder /install` đều **CACHED**. Bắt đầu từ `COPY app`
> thì Docker tạo layer mới (`DONE`); các bước `COPY utils` và `RUN useradd`
> phía sau cũng chạy lại do parent layer đã đổi. Sau phép thử mình bỏ
> comment tạm. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần source
> thay đổi là layer copy mất cache và pip phải cài lại thư viện dù
> `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử một request khai thác lỗi thực thi lệnh trong Python: mã độc sẽ
> chạy với quyền của process trong container. Nếu process là root và
> container còn có cấu hình nguy hiểm như mount `docker.sock` hoặc quyền
> quá rộng, kẻ tấn công có đường leo thang tới host. `USER appuser` trong
> Dockerfile hạ quyền process ngay trước khi chạy app, nên bước đầu tiên
> sau khi khai thác không còn là root. Biện pháp này giảm mức thiệt hại,
> nhưng không thay thế việc bỏ mount/quyền nguy hiểm hay vá lỗ hổng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request**: gửi 10 request lúc 10:00:59 và 10 request lúc
> 10:01:00. Bộ đếm theo phút lịch reset tại giây 00 nên cả hai nhóm đều
> nằm trong hạn mức 10/phút dù cách nhau khoảng một giây. Sliding window
> nhìn 60 giây gần nhất nên nhóm thứ hai sẽ bị chặn sau khi đủ 10 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất** trong 60 giây; cost guard giới hạn
> **tổng chi phí** của từng user trong tháng. Nếu user mới gửi một request
> trong phút này nhưng đã tiêu vượt ngân sách tháng từ lượt trước, rate
> limit cho qua còn cost guard trả 402. Ngược lại, user mới tiêu rất ít
> nhưng gửi hơn 10 request trong 60 giây thì rate limit trả 429 dù ngân
> sách vẫn còn. Trong code hiện tại, `guard.check()` được gọi trước LLM
> nhưng chưa truyền chi phí ước lượng, nên một lượt có thể làm tổng chi
> phí vượt ngưỡng trước khi lượt kế tiếp bị chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu probe chung kiểm tra Redis, khi Redis mất kết nối: (1) cả ba
> container lần lượt báo probe lỗi; (2) orchestrator coi chúng không khỏe,
> rút khỏi nhận traffic và có thể restart cả ba; (3) Redis vẫn lỗi nên
> container mới cũng tiếp tục trượt probe, gây vòng lặp restart; (4) khi
> Redis hồi phục, các instance lại phải khởi động trước khi phục vụ được.
> Với hai endpoint tách biệt, `/health` vẫn 200 để không restart process
> đang sống, còn `/ready` trả 503 để tạm dừng gửi request mới.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy **3 agent container** trong một Compose project tạm, cùng kết nối
> một Redis. Vì Compose gốc map cố định `8000:8000`, mình dùng override tạm
> bỏ cổng host rồi gọi `/ask` trực tiếp bên trong lần lượt replica 1, 2, 3
> với cùng `X-User-Id`. Các giá trị `history_length` thực tế là **0, 2, 4**:
> mỗi lượt lưu thêm một message user và một message assistant, replica sau
> đọc được lịch sử replica trước. Nếu thay Redis bằng dict riêng trong từng
> process, mỗi replica nhận lượt đầu sẽ thấy 0; quay lại replica cũ mới có
> thể thấy 2, nên dãy số sẽ phụ thuộc nơi request được chuyển đến.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy Railway, dashboard từng báo **“Crashed just now”**. Mình kiểm
> tra `railway.toml` và thấy `startCommand` ghi
> `uvicorn ... --port $PORT`; với Dockerfile deployment, lệnh override
> này không mở shell để thay `$PORT`, khiến app không khởi động đúng cổng.
> Mình bỏ `startCommand` để dùng `CMD ["sh", "-c", ...]` trong Dockerfile,
> đồng thời sửa giá trị cấu hình Railway sang dạng hợp lệ, rồi push lại.
> Sau đó app Online; `/health` và `/ready` trả 200, `/ask` thiếu key
> trả 401. Đây là lỗi/nguồn xác định từ dashboard và cấu hình, không phải
> trích dẫn một dòng runtime log cụ thể.
