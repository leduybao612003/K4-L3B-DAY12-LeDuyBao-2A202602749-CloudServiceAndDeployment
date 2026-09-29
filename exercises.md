# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay mỗi dòng trích dẫn placeholder dưới từng câu hỏi bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Duy Bảo  Mã học viên: 2A202602749

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cụ thể: em deploy lên cloud nhưng quên set biến AGENT_API_KEY
> trên dashboard. Vì `agent_api_key` không có mặc định, app ném
> ValidationError ngay lúc khởi động, container crash và log chỉ rõ thiếu
> biến nào nên em sửa được trong vài phút. Nếu để mặc định "changeme",
> app vẫn chạy bình thường, endpoint /ask chấp nhận key "changeme" mà ai
> đọc source cũng đoán được, người lạ gọi chùa và em chỉ phát hiện khi
> nhìn hóa đơn LLM.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật em thu được khi gọi /ask (`docker compose logs agent`):
> `{"event": "ask_completed", "level": "info", "timestamp":
> "2026-09-29T12:56:38.782954+00:00", "user_id": "sv-test", "tokens_in": 3,
> "tokens_out": 37, "cost_usd": 2.265e-05}`. Hai việc làm được mà print
> thường không làm được: (1) gom nhóm theo `user_id` và cộng dồn `cost_usd`
> theo ngày để biết user nào tiêu nhiều tiền nhất; (2) lọc các event lỗi
> theo khung giờ để tính tỉ lệ lỗi 5 phút gần nhất và gắn cảnh báo tự động.
> `print("đã trả lời xong")` không có cấu trúc nên máy không lọc, đếm hay
> cảnh báo được.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Số đo thật trên máy em (`docker images`): bản 1 stage `agent:single`
> chiếm 1.73 GB, bản multi-stage `day12-agent:cp2-test` chiếm 271 MB,
> chênh nhau khoảng 1.46 GB. Phần chênh lệch gồm: base image `python:3.11`
> đầy đủ (Debian nguyên bản kèm build tools) so với `python:3.11-slim` ở
> cả hai stage; pip cache do bản 1 stage cài bằng `pip install` thường còn
> bản multi dùng `--no-cache-dir`; compiler và file trung gian chỉ nằm ở
> stage `builder` rồi bị vứt đi, stage runtime chỉ copy kết quả sang.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile của em (`COPY requirements.txt` rồi `pip install` trước,
> `COPY app`/`COPY utils` sau), sửa một ký tự trong `app/main.py` thì các
> layer từ `COPY requirements.txt` và `pip install` được dùng lại từ cache,
> chỉ các layer từ `COPY app` trở đi phải chạy lại nên build lại rất nhanh.
> Nếu đặt `COPY . .` lên trước `RUN pip install`, mọi lần sửa code đều làm
> thay đổi layer copy, Docker hủy cache từ đó trở đi và phải cài lại toàn
> bộ thư viện — mỗi lần sửa một dấu phẩy cũng build chậm như lần đầu.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: một lỗ hổng trong code Python (ví dụ injection thực thi
> được lệnh hệ thống) cho kẻ tấn công một shell bên trong container; vì
> container chạy mặc định bằng root nên shell đó có uid 0; từ root trong
> container, kẻ tấn công lợi dụng volume mount cấu hình sai hoặc lỗ hổng
> kernel để thoát ra ngoài và có quyền cao trên máy host. Lệnh
> `USER appuser` cắt đứt chuỗi ngay ở mắt xích thứ hai: process chỉ chạy
> với uid 10001 nên dù chiếm được shell, kẻ tấn công cũng không có quyền
> root để leo thang tiếp.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request vào lúc 10:00:59 (cuối phút cũ) rồi gửi
> tiếp 10 request vào lúc 10:01:01 (đầu phút mới), tổng 20 request trong
> 2 giây mà mỗi phút đồng hồ vẫn chỉ ghi nhận 10 nên đều "đúng luật".
> Sliding window 60 giây của em không có kẽ hở này vì nó luôn đếm 60 giây
> gần nhất: tại thời điểm 10:01:01, 10 request lúc 10:00:59 vẫn nằm trong
> cửa sổ nên 10 request mới đã vượt hạn mức và bị trả 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request trong 60 giây (trả 429), còn cost
> guard giới hạn số tiền mỗi user mỗi tháng (trả 402). Rate limit cho qua
> nhưng cost guard chặn: user chỉ gửi 10 request/phút (đúng luật) nhưng mỗi
> request kèm prompt hàng chục nghìn token, tổng tiền vượt 10 USD thì
> `guard.check` chặn 402. Ngược lại, cost guard cho qua nhưng rate limit
> chặn: user spam hàng trăm request rẻ tiền trong một phút, tổng chi phí
> chưa đáng kể nhưng `limiter.check` đã trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện: (1) Redis mất kết nối 30 giây; (2) endpoint gộp kiểm tra
> Redis nên cả 3 container đồng loạt trả lỗi; (3) orchestrator hiểu là cả
> cụm unhealthy và restart cả 3 container cùng lúc; (4) khi Redis quay lại
> thì không còn container nào sống để phục vụ — sự cố nhỏ thành outage toàn
> hệ thống. Tách riêng thì khác: /health (liveness) vẫn 200 nên không ai bị
> restart, còn /ready (readiness) trả 503 để load balancer tạm ngừng đẩy
> traffic vào, hết 30 giây mọi thứ tự hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nói thật kết quả quan sát của em: khi em chạy
> `docker compose up -d --scale agent=3` thì lệnh thất bại với lỗi
> `Bind for 0.0.0.0:8000 failed: port is already allocated` do service
> agent map cố định cổng host 8000, nên em chưa quan sát trực tiếp được 3
> instance cùng chạy. Nhưng theo đúng thiết kế stateless: nếu lịch sử nằm
> trong dict của từng process, `history_length` sẽ nhảy lung tung (lúc 0,
> lúc số cũ) tùy request rơi vào container nào, vì mỗi container có RAM
> riêng; còn lưu trong Redis như bài làm thì con số tăng dần đều 0, 2, 4...
> vì mọi instance cùng đọc một store — điều test `test_lich_su_duoc_dung_lai_giua_cac_request` đã chứng minh.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Em dùng phương án LOCAL_FALLBACK nên không deploy lên cloud; lỗi thật em
> gặp là khi thử scale ở máy: `docker compose up -d --scale agent=3` báo
> `Error response from daemon: ... Bind for 0.0.0.0:8000 failed: port is
> already allocated`. Em tìm nguyên nhân bằng `docker compose ps` (chỉ 1
> container agent Up, 2 container còn lại tạo ra nhưng start thất bại) rồi
> đọc lại `docker-compose.yml` thấy service agent map cố định `8000:8000`.
> Cách sửa: không sửa cấu trúc Compose chỉ để ép lệnh scale chạy được mà
> giữ nguyên 1 instance cho fallback và ghi nhận giới hạn — muốn scale thật
> phải bỏ map cổng host cố định và đặt load balancer (như nginx) phía
> trước, đó là việc để dành khi deploy cloud sau.
