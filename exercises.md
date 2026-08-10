# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Đăng Bách  Mã học viên: 2A202601266


---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên Cloud (Render/Railway), nếu người quản trị quên set biến môi trường `API_TOKEN`:
- Nếu có mặc định ("changeme"): App vẫn khởi động bình thường và trả về status 200. Kẻ tấn công hoặc bot quét tự động phát hiện key mặc định "changeme" và gọi tràn ngập API đến LLM. Bạn chỉ phát hiện ra khi nhận hóa đơn tài chính khổng lồ hoặc hết sạch budget.
- Nếu Fail-fast (không có mặc định): App ném lỗi ValidationError và sập ngay ở bước khởi động (Deploy Failed). Cloud Platform sẽ báo đỏ ngay lập tức trước khi có bất kỳ traffic nào chạm tới app. Bạn sửa lỗi trong 1 phút và tránh hoàn toàn nguy cơ rò rỉ bảo mật cũng như thiệt hại tài chính.


---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log JSON: `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T15:00:00+00:00", "client_id": "sv01", "usd_cost": 0.0001}`
Hai việc làm được:
1. Lọc và nhóm dữ liệu tự động trên Datadog/Grafana (ví dụ: thống kê client_id nào đang tiêu nhiều tiền nhất).
2. Thiết lập Cảnh báo (Alerting) tự động dựa trên giá trị số học của trường (ví dụ: cảnh báo khi usd_cost vượt ngưỡng mà không cần viết regex băm chuỗi).

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
| 1 stage (bản đầu) | 1800 MB |
| Multi-stage | 300 MB |

Giải thích: Phần dung lượng chênh lệch (~1.5GB) chứa trình biên dịch, toolchains (gcc, build-essential), cache installer và bộ SDK Python đầy đủ. Multi-stage build loại bỏ toàn bộ các công cụ build này ở stage `builder` và chỉ copy kết quả thư viện cần thiết sang stage `runtime`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Trả lời:
- Các layer `FROM`, `COPY requirements.txt` và `RUN pip install` được **dùng lại từ cache**. Chỉ từ layer `COPY app ./app` trở đi mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa 1 dòng code, layer `COPY` làm vô hiệu hóa cache (cache invalidation), buộc Docker phải tải và cài lại toàn bộ thư viện pip từ đầu ➔ Làm build chậm đi rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Trả lời:
- Chuỗi sự kiện: Lỗ hổng RCE trong code Python ➔ Kẻ tấn công thực thi lệnh shell trong container ➔ Do container chạy root (UID 0), kẻ tấn công sở hữu quyền root ➔ Khai thác lỗ hổng thoát container (container escape) để chiếm toàn bộ máy host.
- Lệnh `USER appuser` chuyển container sang chạy bằng người dùng không có quyền quản trị (UID 10001) ➔ Cắt đứt chuỗi tấn công ngay từ bước 3 (kẻ tấn công không có quyền root nên không thể thực hiện các thao tác độc hại lên kernel/host).


---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

Trả lời:
- 401 phải kèm `WWW-Authenticate: Bearer` vì đây là quy chuẩn RFC 6750 HTTP bắt buộc, thông báo cho client biết phương thức xác thực chuẩn mà server yêu cầu.
- Trả cùng một thông báo lỗi để tránh rò rỉ thông tin cho kẻ tấn công (enumeration attack). Nếu phân biệt "sai scheme" hay "sai token", kẻ tấn công sẽ biết từng bước mình đã đoán đúng phần nào để tiếp tục dò tìm.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Trả lời:
- Gửi được tối đa **10 request** trước khi bị 429 (vì capacity chốt tối đa là 10 token).
- Nếu bỏ `min(capacity, ...)`: Sau 10 phút im lặng, số token tích lũy sẽ thành `10 + 10 * 10 = 110 token`. Client có thể bắn dồn dập **110 request** trong 1 giây, làm vô hiệu hóa khả năng kiểm soát lưu lượng tức thời của rate limiter.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Trả lời:
- Hạn mức $30/tháng: Thiệt hại tối đa là **$30** (bị đốt sạch chỉ trong vài giờ đầu tiên). Service bị dừng hoàn toàn cho đến hết tháng (30 ngày sau).
- Hạn mức $1/ngày: Thiệt hại tối đa bị chặn ở **$1**. Sang ngày hôm sau (00:00 UTC), key Redis theo ngày tự làm mới và service **tự động khôi phục** hoạt động mà không cần can thiệp.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện khi gộp endpoint:
1. Giây 0: Redis mất kết nối ➔ Endpoint gộp trả về 503 cho cả liveness và readiness.
2. Giây 10: Orchestrator tưởng process của cả 3 container đã chết/treo ➔ Đồng loạt **kill và restart** cả 3 container.
3. Giây 15–25: 3 container mới khởi động lại nhưng Redis vẫn sập ➔ Tiếp tục trả 503 ➔ Orchestrator lại tiếp tục kill & restart (CrashLoopBackOff).
4. Giây 30: Redis phục hồi trở lại, nhưng cả cụm 3 container đang sập và trong quá trình restart lại từ đầu ➔ Hệ thống bị gián đoạn hoàn toàn (cascading failure).
(Tách riêng: `/healthz` giữ container không bị restart oan, `/readyz` chỉ tạm ngắt traffic từ Load Balancer).


---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Trả lời:
- Lỗi gặp phải: Health Check Timeout (Lỗi 502 Bad Gateway / Container Healthcheck Failed) khi mới deploy ứng dụng lên Cloud Platform.
- Nguyên nhân: Ứng dụng Uvicorn cố định bind địa chỉ `127.0.0.1:8000` thay vì `0.0.0.0:${PORT}`. Trên Cloud Platform (Render/Railway), cổng ứng dụng được gán động qua biến môi trường `$PORT`, và Load Balancer yêu cầu ứng dụng phải lắng nghe trên giao diện mạng `0.0.0.0`.
- Cách tìm nguyên nhân: Kiểm tra Runtime Logs trên Dashboard của Cloud Platform, thấy ứng dụng khởi tạo ở `127.0.0.1:8000` nhưng liveness check của Cloud bị ngắt kết nối (Connection Refused).
- Cách khắc phục: Cập nhật lệnh `CMD` trong Dockerfile để đọc động cổng `$PORT` và bind địa chỉ `0.0.0.0`:
  `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`.

