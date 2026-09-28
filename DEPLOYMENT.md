# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Tạ Hoàng Vinh |
| Mã học viên | 2A202602543 |
| Repo | https://github.com/nguoibian863-ai/K4-L3A-TaHoangVinh-2A202602543-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3a-tahoangvinh-2a202602543-cloud.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | biến môi trường trên Render dashboard |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://k4-l3a-tahoangvinh-2a202602543-cloud.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://k4-l3a-tahoangvinh-2a202602543-cloud.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://k4-l3a-tahoangvinh-2a202602543-cloud.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://k4-l3a-tahoangvinh-2a202602543-cloud.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://k4-l3a-tahoangvinh-2a202602543-cloud.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
1. GET /health
HTTP/1.1 200 OK
content-length: 50
content-type: application/json

{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. GET /ready
HTTP/1.1 200 OK
content-length: 31
content-type: application/json

{"status":"ready","redis":true}

3. POST /ask (không có API key)
HTTP/1.1 401 Unauthorized
content-length: 42
content-type: application/json

{"detail":"invalid or missing API key"}

4. POST /ask (có API key)
HTTP/1.1 200 OK
content-length: 224
content-type: application/json

{"answer":"Deploy là quá trình đóng gói và phát hành ứng dụng lên môi trường máy chủ để người dùng có thể truy cập.","user_id":"sv-test","history_length":0,"cost_usd":0.00012,"tokens":{"in":16,"out":45}}

5. Rate limit test
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

Chưa có thẻ thanh toán quốc tế và tài khoản Railway hết hạn mức credit dùng thử miễn phí, nên sử dụng phương án dự phòng LOCAL_FALLBACK=true kết hợp Docker Compose stack cục bộ và chụp ảnh minh chứng trong thư mục screenshots/.
