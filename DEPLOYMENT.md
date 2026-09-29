# Thông Tin Deploy — Checkpoint 5 (phương án LOCAL_FALLBACK)

> Phiên hỗ trợ này chỉ thực hiện phương án dự phòng `LOCAL_FALLBACK=true`:
> stack chạy bằng `docker compose` ở máy học viên, kiểm tra ở
> `http://localhost:8000`. Không deploy Railway/Render, không có Public URL.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Lê Duy Bảo |
| Mã học viên | 2A202602749 |
| Repo | https://github.com/leduybao612003/K4-L3B-DAY12-LeDuyBao-2A202602749-CloudServiceAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | chưa deploy lên cloud |
| URL kiểm tra local | http://localhost:8000 |
| Platform | local fallback ở máy — chưa deploy lên Railway hay Render |
| Ngày kiểm tra | 2026-09-29 |

## Biến Môi Trường Đã Set ở local

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | 8000, map `8000:8000` trong docker-compose.yml |
| `AGENT_API_KEY` | ✅ | đặt trong file `.env` |
| `REDIS_URL` | ✅ | `redis://redis:6379/0` trong service `agent` của docker-compose.yml, trỏ tới service `redis` trong cùng compose |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |
| `LOCAL_FALLBACK` | ✅ | `true` trong `.env` cục bộ để test CP5 chuyển sang chế độ kiểm tra local |

## Lệnh Kiểm Tra

Chạy với service local ở `http://localhost:8000`.

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i http://localhost:8000/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i http://localhost:8000/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Xem trạng thái stack
docker compose ps
docker compose logs agent --tail 20
```

## Kết Quả Chạy Thật

Xác minh ngày 2026-09-29 sau `docker compose up -d`

```
GET  /health          -> 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready           -> 200 {"status":"ready","redis":true}
POST /ask (no key)    -> 401
POST /ask (sai key)   -> 401
POST /ask (đúng key)  -> 200, user_id "sv-test", có trường answer
```

`docker compose ps` thấy 2 container `Up`: service `agent` map
`0.0.0.0:8000->8000/tcp` và chuyển sang trạng thái healthy, service `redis`
healthy. Log của `agent` ghi nhận `GET /health 200`, `GET /ready 200`,
`POST /ask 401` khi thiếu key và một dòng JSON
`{"event": "ask_completed", ... "user_id": "sv-test", ...}` kèm
`POST /ask 200` khi có key hợp lệ.

## Ảnh Chụp Màn Hình

Thư mục `screenshots/`:

- `screenshots/compose_ps.png` — 
- `screenshots/health.png` 

---
## Phương Án Dự Phòng Đang Dùng

1. Đã đặt `LOCAL_FALLBACK=true` trong `.env` cục bộ, file này được git ignore nên không vào repo.
2. Đã chạy `docker compose up -d` và kiểm tra `docker compose ps`.
3. Ảnh chụp màn hình: trong folder `screenshots/`
4. `pytest tests/test_cp5.py -v` tự chuyển sang kiểm tra `http://localhost:8000`.
5. Lý do dùng phương án dự phòng: chưa biết deploy lên railway
