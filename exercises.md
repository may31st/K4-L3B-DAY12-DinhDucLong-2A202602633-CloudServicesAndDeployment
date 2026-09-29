# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời trực tiếp bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đinh Đức Long  Mã học viên: 2A202602633

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên môi trường staging hoặc production, nếu người vận hành quên cấu hình biến `AGENT_API_KEY` mà ứng dụng lại có giá trị mặc định `"changeme"`, service vẫn sẽ khởi động bình thường và vượt qua các bài kiểm tra liveness ban đầu. Khi đó, bất kỳ ai (kể cả kẻ tấn công) cũng có thể thử dùng khóa mặc định `"changeme"` để truy cập API `/ask`, gây rò rỉ dữ liệu và làm cạn kiệt ngân sách gọi model LLM. Ngược lại, nhờ cơ chế "fail fast", ứng dụng lập tức crash ngay khi khởi động (ValidationError), giúp orchestrator (Docker/Kubernetes/Railway) phát hiện lỗi cấu hình ngay tại thời điểm deploy và không đưa bản deploy lỗi ra phục vụ người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"timestamp": "2026-09-29T07:47:45.123456Z", "level": "INFO", "message": "request processed", "user_id": "sv-test", "cost_usd": 0.0000957, "duration_ms": 32.5, "status_code": 200}
```

Hai việc làm được với structured log JSON mà `print` thông thường không thể làm được:
1. **Truy vấn và lọc theo cấu trúc (Structured Querying):** Các hệ thống thu thập log tập trung (Datadog, Grafana Loki, CloudWatch, ELK) có thể tự động bóc tách từng trường để lọc nhanh tất cả request của một `user_id` nhất định hoặc tìm các request có độ trễ cao (`duration_ms > 1000`) mà không cần viết các câu lệnh Regular Expression phức tạp.
2. **Tổng hợp số liệu và cảnh báo tự động (Aggregation & Alerting):** Có thể tính toán tổng chi phí tiêu thụ `sum(cost_usd)` theo thời gian thực hoặc xây dựng biểu đồ độ trễ p95/p99 từ trường số nguyên `duration_ms` để phát hiện bất thường và bắn cảnh báo ngay khi chi phí vượt ngưỡng an toàn.

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
| 1 stage (bản đầu) | ~385 MB |
| Multi-stage | ~175 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~210 MB) bao gồm:
- Các công cụ biên dịch (compiler toolchain) như `gcc`, `g++`, `make`, `python3-dev` và các header C cần dùng trong giai đoạn `builder` để compile bánh xe wheel/C extension cho thư viện.
- Bộ nhớ cache của package manager (`~/.cache/pip`, `/var/cache/apt`, `/var/lib/apt/lists/*`) sinh ra trong quá trình cài đặt package.
- Các file tạm, tài liệu hướng dẫn (manpages), và file test thừa của các dependencies. Stage cuối (`final`) chỉ sao chép đúng thư mục thư viện đã cài đặt sẵn (`site-packages`) và mã nguồn app vào một base image `python:3.11-slim` hoàn toàn sạch.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Nhờ copy riêng `requirements.txt` trước khi chạy `pip install`, khi sửa một ký tự trong `app/main.py`, file `requirements.txt` không thay đổi nên Docker tái sử dụng (CACHE) toàn bộ các layer tải base image, cài đặt hệ thống và cài đặt thư viện (`RUN pip install`). Chỉ có layer `COPY app/ ./app` và các bước kế sau phải chạy lại, quá trình build chỉ mất chưa đầy 1 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ file mã nguồn nào, checksum của thư mục thay đổi sẽ làm mất hiệu lực (bust cache) của layer `COPY . .`. Do đó, Docker sẽ buộc phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu, kéo dài thời gian build lên vài phút và lãng phí băng thông mạng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Ứng dụng Python có lỗ hổng bảo mật (ví dụ: Command Injection, Insecure Deserialization, hoặc RCE từ thư viện bên thứ ba).
2. Kẻ tấn công kích hoạt lỗ hổng để thực thi shell command trong container. Nếu container chạy với người dùng mặc định (root - UID 0), tiến trình của kẻ tấn công nắm toàn quyền root trong namespace của container.
3. Kẻ tấn công tận dụng các cấu hình mount nguy hiểm (như `/var/run/docker.sock`, mount ổ đĩa host) hoặc lỗ hổng kernel / container escape (như các lỗ hổng của `runc`) để thoát ra ngoài container. Vì tiến trình mang UID 0, khi thoát ra máy host nó vẫn mang quyền root của máy host, dẫn đến chiếm toàn quyền máy chủ.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay tại bước 2: Tiến trình chỉ chạy dưới một user thường không có đặc quyền (non-root UID 10001). Kẻ tấn công bị chặn quyền can thiệp vào các file nhạy cảm của hệ điều hành, không có quyền quản trị và không có các Linux Capabilities (`CAP_SYS_ADMIN`) cần thiết để thực hiện container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Cách đạt được:
- Tại giây thứ 59 của phút trước: người dùng gửi dồn dập 10 request. Hệ thống fixed window đếm đủ 10/10 request trong phút đó.
- Ngay khi đồng hồ chuyển sang giây thứ 00 của phút kế tiếp: bộ đếm của phút cũ được reset về 0.
- Tại đúng giây thứ 00 này, người dùng gửi tiếp 10 request nữa và hệ thống vẫn chấp nhận toàn bộ vì bộ đếm phút mới mới chỉ ghi nhận 10 request.
Như vậy, từ giây 59 đến giây 00 (khoảng thời gian chỉ 2 giây), hệ thống đã phải chịu tải 10 + 10 = 20 request (gấp đôi hạn mức). Sliding window (cửa sổ trượt) giải quyết triệt để lỗi này bằng cách xét số lượng request trong đúng 60 giây trôi qua tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau:** Rate limit kiểm soát **tốc độ / tần suất** request trong khoảng thời gian ngắn (ví dụ: 10 request / 60 giây) nhằm bảo vệ hạ tầng máy chủ khỏi quá tải và DDoS. Cost guard kiểm soát **tổng chi phí tài chính / hạn mức ngân sách** trong chu kỳ dài (ví dụ: $5.0 / tháng) nhằm bảo vệ tiền bạc trước rủi ro chi phí LLM tăng vọt.
- **Rate limit cho qua nhưng Cost guard chặn:** Người dùng chỉ gửi 1 request trong vòng cả tiếng đồng hồ (tốc độ hoàn toàn bình thường, rate limit cho qua). Tuy nhiên tài khoản của người này trong tháng đã tiêu hết hạn mức $5.0. Cost guard sẽ chặn ngay lập tức và trả về mã lỗi `402 Payment Required`.
- **Cost guard cho qua nhưng Rate limit chặn:** Người dùng mới sử dụng dịch vụ, chi phí tích lũy trong tháng mới chỉ là $0.001 (rất thấp so với ngân sách $5.0). Tuy nhiên người dùng dùng script gửi liên tiếp 25 request trong 3 giây. Cost guard không kích hoạt, nhưng Rate limiter lập tức chặn từ request thứ 11 trở đi và trả về mã lỗi `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis gặp sự cố hoặc gián đoạn mạng, mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra thấy Redis không kết nối được nên trả về mã lỗi 503 (hoặc timeout).
3. Orchestrator (Docker / Kubernetes) định kỳ gọi Liveness probe kiểm tra container. Thấy endpoint trả về lỗi liên tiếp, orchestrator kết luận rằng cả 3 container agent đều đã bị treo tiến trình (unhealthy).
4. Orchestrator lập tức cưỡng chế khởi động lại (restart / kill) đồng loạt cả 3 container.
5. Cả 3 container mới khởi động lại, nhưng vì Redis vẫn chưa hồi phục nên khi probe kiểm tra, chúng lại tiếp tục báo lỗi.
6. Hệ thống rơi vào vòng lặp khởi động lại liên tục (CrashLoopBackOff). Cụm ứng dụng bị gián đoạn hoàn toàn (cascading failure), thay vì vẫn giữ các container sống để trả lời các endpoint tĩnh và chờ Redis hoạt động trở lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trên Redis (chuẩn Stateless): Biến `history_length` tăng đều đặn và nhất quán (0 -> 2 -> 4 -> 6...) bất kể request được điều phối tới container nào trong cụm 3 instance, vì cả 3 instance cùng đọc/ghi vào một Redis trung tâm.
- Nếu lưu trong dict Python (Stateful / in-memory): Do Load Balancer phân phối các request ngẫu nhiên hoặc xoay vòng (round-robin) qua 3 container A, B, C độc lập:
  - Request 1 rơi vào A -> RAM của A lưu 1 lượt trao đổi -> `history_length` = 0 (sau đó thành 2).
  - Request 2 rơi vào B -> RAM của B chưa có dữ liệu user này -> `history_length` bị reset về 0.
  - Request 3 rơi vào C -> `history_length` tiếp tục là 0.
  - Request 4 rơi lại vào A -> `history_length` lúc này mới tăng lên 2.
  Con số `history_length` sẽ nhảy lộn xộn, trồi sụt bất thường phụ thuộc vào container nào tiếp nhận request, phá vỡ tính liên tục của ngữ cảnh hội thoại AI.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi phân giải cổng và cú pháp khi gửi body JSON qua lệnh curl trên terminal Windows PowerShell (`422 Unprocessable Entity` và `curl: (3) URL rejected: Port number was not a decimal number between 0 and 65535`).
- **Nguyên nhân:** Trên Windows PowerShell, khi gõ lệnh curl trực tiếp với tham số chuỗi kép `-d "{\"question\":\"...\"}"`, parser của PowerShell tự động bóc các dấu ngoặc kép trước khi truyền xuống cho `curl.exe`. Dấu hai chấm `:` trong chuỗi JSON bị `curl.exe` hiểu nhầm thành ký tự phân tách số cổng của URL, đồng thời nội dung JSON bị cắt rời rạc gây lỗi giải mã tại endpoint FastAPI.
- **Cách tìm ra và khắc phục:** Đọc kỹ thông báo lỗi chi tiết `JSON decode error: Expecting property name enclosed in double quotes` từ phản hồi của server; sau đó chuẩn hóa cú pháp trên PowerShell bằng cách sử dụng pipeline để truyền trực tiếp chuỗi qua stdin: `'{"question":"Deploy la gi?"}' | curl.exe -i -X POST ... -d "@-"`. Cách này đảm bảo chuỗi JSON được giữ nguyên vẹn 100% khi gửi lên production server trên Railway.
