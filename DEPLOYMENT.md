# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Hoàng Công Minh |
| Mã học viên | 2A202602774 |
| Repo | https://github.com/congminh1705/K4-L3A-DAY12-HoangCongMinh-2A202602774-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3a-day12-hoangcongminh-2a202602774-cloudserv-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 28/09/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | tham chiếu tới service Redis trên Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | cấu hình trên Railway |
| `MONTHLY_BUDGET_USD` | ✅ | cấu hình trên Railway |
| `LOG_LEVEL` | ✅ | cấu hình trên Railway |

## Lệnh Kiểm Tra

Các lệnh dưới đây dùng Public URL ở trên. Lệnh có API key và kiểm tra rate limit
là lệnh tham khảo, chưa được ghi nhận là đã chạy trong phần kết quả bên dưới.

```bash
URL=https://k4-l3a-day12-hoangcongminh-2a202602774-cloudserv-production.up.railway.app

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i "$URL/health"

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i "$URL/ready"

# 3. Không có API key — mong đợi 401
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST "$URL/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST "$URL/ask" \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

### `/health` — HTTP 200

```bash
curl -i https://k4-l3a-day12-hoangcongminh-2a202602774-cloudserv-production.up.railway.app/health
```

```text
HTTP/2 200
content-type: application/json
date: Mon, 28 Sep 2026 10:08:52 GMT
server: railway-hikari
x-railway-request-id: UE-SuqxTQHikTxkzlt7tkg
content-length: 57
x-hikari-trace: hkg1.aebn
x-railway-edge: hkg1

{"status":"ok","service":"day12-agent","version":"1.0.0"}
```

Xem ảnh `screenshots/health.png` có URL và phản hồi trong cùng khung hình.

### `/ready` — HTTP 200

```bash
curl -i https://k4-l3a-day12-hoangcongminh-2a202602774-cloudserv-production.up.railway.app/ready
```

```text
HTTP/2 200
content-type: application/json
date: Mon, 28 Sep 2026 10:08:50 GMT
server: railway-hikari
x-railway-request-id: nyyyqKoySouQuOAGWUN5dQ
content-length: 31
x-hikari-trace: hkg1.hn7d
x-railway-edge: hkg1

{"status":"ready","redis":true}
```

### `/ask` không có API key — HTTP 401

```bash
curl -i -X POST https://k4-l3a-day12-hoangcongminh-2a202602774-cloudserv-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

```text
HTTP/2 401
content-type: application/json
date: Mon, 28 Sep 2026 10:08:51 GMT
server: railway-hikari
x-railway-request-id: WBUB-g5HTS6nNvMGwUFZXw
content-length: 39
x-hikari-trace: hkg1.hn7d
x-railway-edge: hkg1

{"detail":"invalid or missing API key"}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
