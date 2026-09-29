# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đinh Đức Long |
| Mã học viên | 2A202602633 |
| Repo | https://github.com/may31st/K4-L3B-DAY12-DinhDucLong-2A202602633-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-3174.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của platform (`day12-redis`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

### Trên Linux / macOS (Bash)

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-3174.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-production-3174.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-3174.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-production-3174.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-production-3174.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

### Trên Windows (PowerShell)

> **Lưu ý:** Trên Windows PowerShell, sử dụng `curl.exe` thay cho `curl` để tránh xung đột alias với cmdlet `Invoke-WebRequest`.

```powershell
# 1. Liveness
curl.exe -i https://day12-agent-production-3174.up.railway.app/health

# 2. Readiness
curl.exe -i https://day12-agent-production-3174.up.railway.app/ready

# 3. Không có API key
'{"question":"Hello"}' | curl.exe -i -X POST https://day12-agent-production-3174.up.railway.app/ask -H "Content-Type: application/json" -d "@-"

# 4. Có API key
$key = (Get-Content .env | Select-String "^AGENT_API_KEY=").ToString().Split("=")[1].Trim()
'{"question":"Deploy la gi?"}' | curl.exe -i -X POST https://day12-agent-production-3174.up.railway.app/ask -H "Content-Type: application/json" -H "X-API-Key: $key" -H "X-User-Id: sv-test" -d "@-"

# 5. Rate limit (15 lần)
1..15 | ForEach-Object {
  '{"question":"test"}' | curl.exe -s -o NUL -w "%{http_code} " -X POST https://day12-agent-production-3174.up.railway.app/ask -H "Content-Type: application/json" -H "X-API-Key: $key" -H "X-User-Id: sv-test-rate" -d "@-"
}; ""
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 29 Sep 2026 07:41:30 GMT
Server: railway-hikari
x-railway-request-id: aWOrEsyEQ02q-ewmY53eZw
Content-Length: 57
x-hikari-trace: sin1.d1nj
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 29 Sep 2026 07:40:52 GMT
Server: railway-hikari
x-railway-request-id: xqGWFHDcTby45sDPY53eZw
Content-Length: 31
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"status":"ready","redis":true}

# 3. Không có API key
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Tue, 29 Sep 2026 07:47:10 GMT
Server: railway-hikari
x-railway-request-id: tzFOPUteQF-lt9-bn6XIxQ
Content-Length: 39
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
Connection: keep-alive

{"detail":"invalid or missing API key"}

# 4. Có API key
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 29 Sep 2026 07:47:45 GMT
Server: railway-hikari
x-railway-request-id: Brp0JuFtRGC0dB082prcFg
Content-Length: 347
x-hikari-trace: sin1.nzn2
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên. (Mình đang nhớ 18 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":18,"cost_usd":9.57e-05,"tokens":{"in":450,"out":47}}

# 5. Rate limit
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429 
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
