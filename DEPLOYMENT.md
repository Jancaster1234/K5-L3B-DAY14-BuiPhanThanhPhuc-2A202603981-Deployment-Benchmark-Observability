# Deployment Information

## Public URL
https://agent-production-9e5a9744.up.railway.app

## Platform
Railway: project `k5-day14-agent`, environment `production`, region `iad`.

Hai service:
- **`agent`**: build từ `06-lab-complete/Dockerfile` theo `06-lab-complete/railway.toml` (`builder = "DOCKERFILE"`, `healthcheckPath = "/health"`). Deploy bằng `railway up --service agent` từ thư mục `06-lab-complete/`.
- **`Redis`**: database Redis của Railway (`railway add --database redis`). Agent kết nối qua mạng nội bộ `redis.railway.internal`.

## Test Commands
Đặt `BASE` là URL ở trên và `AGENT_API_KEY` là key đã đặt trên Railway (không ghi trong repo):

```bash
BASE=https://agent-production-9e5a9744.up.railway.app
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả chạy lúc 23:36 ngày 2026-11-26:
```text
health: 200
ready:  200
no key: 401
key:    200
```

Câu trả lời mẫu (có key):
```json
{"question":"What is docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

Script chấm kiểm URL public:
```text
$ python check_production_ready.py --public-url https://agent-production-9e5a9744.up.railway.app
✅ Public URL: GET https://agent-production-9e5a9744.up.railway.app/health → 200
```

## Environment Variables Set
Đặt trên Railway (service `agent` → Variables), **không** ghi trong code hay repo:

| Biến | Giá trị |
|---|---|
| `AGENT_API_KEY` | chuỗi ngẫu nhiên 32 ký tự (`secrets.token_urlsafe(24)`), giữ bí mật |
| `REDIS_URL` | `${{Redis.REDIS_URL}}`: biến tham chiếu của Railway, tự trỏ tới service Redis |
| `ENVIRONMENT` | `production` |
| `RATE_LIMIT_PER_MINUTE` | `10` |
| `DAILY_BUDGET_USD` | `1.0` |
| `MONTHLY_BUDGET_USD` | `10.0` |
| `PORT` | Railway tự cấp; `CMD` trong Dockerfile đọc `${PORT:-8000}` |

## Screenshots
- `screenshots/dashboard.png`: dashboard Railway với 2 service `agent` + `Redis`
- `screenshots/running.png`: deployment của `agent` đang chạy (Active)
- `screenshots/test.png`: kết quả 4 lệnh test ở trên
