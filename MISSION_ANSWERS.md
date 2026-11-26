# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
Trong `01-localhost-vs-production/develop/app.py`:
1. **Hardcode secret:** `OPENAI_API_KEY = "sk-hardcoded-..."` và `DATABASE_URL = "postgresql://admin:password123@..."`. Push lên GitHub là lộ key ngay, và key vẫn nằm trong lịch sử git kể cả sau khi xoá.
2. **Log ra secret:** `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`. Ai đọc log là có key.
3. **`print` thay vì logging có cấu trúc:** không có level, không có timestamp, không parse được bằng log aggregator.
4. **Không có config management:** `DEBUG = True`, `MAX_TOKENS = 500` cố định trong code; đổi môi trường phải sửa code.
5. **Không có health check:** `GET /health` trả 404, platform không biết agent còn sống để restart.
6. **Bind `host="localhost"`:** chạy trong container thì không nhận được kết nối từ bên ngoài.
7. **Port cố định `8000`:** Railway/Render cấp port qua biến `PORT`, app không đọc nên sẽ không nhận traffic.
8. **`reload=True`:** chế độ dev, tốn tài nguyên và không an toàn khi chạy production.
9. **Không có graceful shutdown:** nhận SIGTERM là tắt ngay, request đang xử lý bị cắt ngang.

### Exercise 1.2: Kết quả gọi bản develop
```text
$ curl -X POST "http://localhost:8011/ask?question=Hello"
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
$ curl -o /dev/null -w "%{http_code}" http://localhost:8011/health
404
```
(Chạy bằng `uvicorn app:app --port 8011` vì cổng 8000 trên máy đang bận.) App trả lời được, nhưng chạy được trên laptop không có nghĩa là sẵn sàng cho production: không có `/health`, secret nằm trong code và bị in ra log.

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trong code (`DEBUG = True`, key, DB URL) | `config.py` đọc từ env (`os.getenv("PORT")`, `OPENAI_API_KEY`, `DEBUG`…) | Cùng một image chạy được ở dev/staging/prod; đổi cấu hình không cần sửa code (12-Factor) |
| Secrets | `sk-...` và mật khẩu DB nằm trong code, còn bị in ra log | Đọc từ env, không log ra | Lộ key trên GitHub là bị dùng trộm, mất tiền ngay |
| Logging | `print(...)` | JSON log có level/time (`logging` + format JSON) | Log aggregator (Loki, Datadog) lọc/tìm được; không lộ dữ liệu nhạy cảm |
| Health check | Không có (`/health` → 404) | `GET /health` (liveness) + `GET /ready` (503 khi chưa sẵn sàng) | Platform biết khi nào restart container, load balancer biết khi nào gửi traffic |
| Host / Port | `localhost:8000` cố định | `HOST=0.0.0.0`, `PORT` từ env | Container và cloud platform cần bind 0.0.0.0 và port do platform cấp |
| Shutdown | Tắt ngay | Lifespan + handler SIGTERM, xử lý xong request đang chạy | Deploy/scale down không làm hỏng request của người dùng |
| Reload | `reload=True` | Không reload | Reload chỉ để dev, tốn tài nguyên và không ổn định |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)
1. **Base image:** `python:3.11`, bản Python đầy đủ (có compiler, header, công cụ build), nặng gần 1 GB.
2. **Working directory:** `WORKDIR /app`. Mọi lệnh `COPY`/`RUN`/`CMD` sau đó chạy trong `/app`.
3. **Vì sao `COPY requirements.txt` trước rồi mới `COPY app.py`:** tận dụng layer cache. Docker build lại từ layer đầu tiên bị thay đổi trở đi. Code đổi thường xuyên, requirements ít đổi; copy requirements và `pip install` trước thì sửa code chỉ build lại layer copy code (vài giây), không phải cài lại dependencies.
4. **CMD vs ENTRYPOINT:** `CMD ["python", "app.py"]` là lệnh mặc định, bị thay thế hoàn toàn khi chạy `docker run image <lệnh khác>`. `ENTRYPOINT` là phần cố định luôn chạy, tham số của `docker run` được nối thêm vào sau. Dùng exec form (`[...]`) để process là PID 1 và nhận được SIGTERM.

### Exercise 2.2: Chạy container develop
```text
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -d -p 8031:8000 my-agent:develop
$ curl -X POST "localhost:8031/ask?question=What%20is%20Docker"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

### Exercise 2.3: Image size comparison
Đo bằng `docker images` (dung lượng image giải nén trên đĩa):
- Develop (`my-agent:develop`, single-stage `python:3.11`): **1.67 GB ≈ 1670 MB**
- Production (`my-agent:advanced`, multi-stage `python:3.11-slim`): **240 MB**
- Difference: **giảm ≈ 85.6%** ((1670 − 240) / 1670)

(`docker image inspect -f '{{.Size}}'` báo dung lượng nén: 426,319,957 byte ≈ 407 MiB so với 57,409,026 byte ≈ 55 MiB, giảm 86.5%. Hai cách đo cho cùng kết luận.)

**Multi-stage giảm dung lượng ở đâu:** stage `builder` cài `gcc`, `libpq-dev` và build các package; stage `runtime` bắt đầu lại từ `python:3.11-slim` sạch và chỉ `COPY --from=builder /root/.local`, tức chỉ lấy package đã cài. Compiler, header, cache apt nằm lại ở builder, không vào image cuối. Image production còn chạy bằng user non-root `appuser` và có `HEALTHCHECK`.

### Exercise 2.4: Docker Compose stack
Sơ đồ `02-docker/production/docker-compose.yml`:
```text
client ──▶ nginx :80/:443 (reverse proxy, upstream agent_backend)
              │
              ▼
           agent :8000 (build target "runtime", uvicorn --workers 2, healthcheck /health)
              │                         │
              ▼                         ▼
     redis:7-alpine :6379        qdrant:v1.9.0 :6333
     (cache / session)           (vector DB)
     ── tất cả trong network nội bộ "internal"; chỉ nginx mở cổng ra ngoài ──
```
- `agent` có `depends_on` redis và qdrant với `condition: service_healthy`, nên chỉ khởi động khi hai service kia đã healthy.
- Secret đọc từ `env_file: .env.local` (nằm trong `.gitignore`).
- Nginx: `location /` và `/health` đều `proxy_pass http://agent_backend`; client không gọi thẳng agent được.

_Chưa chạy được stack 2.4 trước hạn nộp: Docker Hub đang giới hạn tốc độ pull (lỗi 429) ở máy mình, và stack cần thêm `qdrant`, `nginx:alpine`. Phần trên mô tả từ file cấu hình, chưa có output test qua Nginx._

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy agent Part 6
- URL: _TODO_
- Screenshot: `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png`

### So sánh `railway.toml` và `render.yaml`
So sánh hai file trong `06-lab-complete/`:

| | `railway.toml` | `render.yaml` |
|---|---|---|
| Phạm vi | Chỉ cấu hình **một service** (cách build + deploy) | **Blueprint cả hạ tầng**: service web + Redis (`type: keyvalue`) trong một file |
| Build | `builder = "DOCKERFILE"`, `dockerfilePath` | `runtime: docker`, `rootDir: 06-lab-complete`, `dockerfilePath` (file nằm trong thư mục con nên phải chỉ `rootDir`) |
| Health check | `healthcheckPath = "/health"`, `healthcheckTimeout` | `healthCheckPath: /health` |
| Restart | `restartPolicyType = "ON_FAILURE"`, số lần retry | Render tự quản lý, không khai báo |
| Biến môi trường | **Không có trong file**, đặt bằng dashboard/CLI (`railway variables --set ...`) | Khai báo ngay trong file: giá trị thường ghi thẳng; secret `AGENT_API_KEY` dùng `generateValue: true` (Render tự sinh); `REDIS_URL` lấy bằng `fromService` từ Redis |
| Redis | Tạo Redis riêng trên Railway, trỏ `REDIS_URL='${{Redis.REDIS_URL}}'` | Khai báo service `keyvalue` và nối tự động qua `fromService` |
| Port | Railway cấp biến `PORT`, `CMD` của Dockerfile đọc `${PORT:-8000}` | Giống vậy |

**Vì sao khác nhau:** Railway xem file config là "cách build/chạy một service", còn hạ tầng (Redis, biến) tạo trên dashboard. Render theo kiểu Infrastructure-as-Code: một Blueprint mô tả cả hệ thống, nên tạo lại được toàn bộ từ repo. Cả hai đều không được ghi secret thật vào file.

### Exercise 3.3: Pipeline Cloud Run
`cloudbuild.yaml` chạy 4 bước nối tiếp, bước sau chỉ chạy khi bước trước thành công (`waitFor`):
1. **test:** trong `python:3.11-slim`, cài `requirements.txt` + pytest và chạy `pytest tests/`. Test fail là dừng pipeline.
2. **build:** `docker build` image, gắn 2 tag `:$COMMIT_SHA` và `:latest`, dùng `--cache-from :latest` để tận dụng layer cache.
3. **push:** đẩy image lên Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **deploy:** `gcloud run deploy` image theo đúng `$COMMIT_SHA` (truy vết được bản nào đang chạy) ở `asia-southeast1`, `min-instances=1` để tránh cold start, `max-instances=10`, 512Mi/1 CPU, timeout 60 s; `ENVIRONMENT` qua `--set-env-vars`, `OPENAI_API_KEY` lấy từ Secret Manager (`--set-secrets`), không nằm trong code.

`service.yaml` mô tả cùng service đó theo kiểu khai báo (Knative `serving.knative.dev/v1`): `minScale 1` / `maxScale 10`, `containerConcurrency: 80` (mỗi instance nhận tối đa 80 request đồng thời, vượt thì scale thêm), request 0.5 CPU / 256Mi, limit 1 CPU / 512Mi, `containerPort: 8000`. Áp dụng bằng `gcloud run services replace service.yaml`, để quản lý cấu hình bằng file trong git thay vì cờ dòng lệnh.

## Part 4: API Security

### Exercise 4.1-4.3: Test results

**4.1 — API key (bản develop, `AGENT_API_KEY=secret-key-123`):**
```text
$ curl -i -X POST localhost:8021/ask -H 'Content-Type: application/json' -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

$ curl -i -X POST localhost:8021/ask -H 'X-API-Key: secret-key-123' -H 'Content-Type: application/json' -d '{"question":"Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}

$ curl -i -X POST localhost:8021/ask -H 'X-API-Key: wrong' -H 'Content-Type: application/json' -d '{"question":"Hello"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid API key."}
```
Bản develop phân biệt **thiếu** key (401) và **sai** key (403). Agent Part 6 của mình trả 401 cho cả hai, theo hợp đồng của script chấm. Lưu ý: bản develop Part 4 đọc `question` từ body JSON; gửi `?question=` thì body rỗng và app trả 500.

**4.2 — JWT (bản production):** lấy token bằng `POST /auth/token` với `{"username":"student","password":"demo123"}`, nhận `access_token` `eyJhbGciOiJI…` (đã che). Gọi `/ask` không có token:
```text
HTTP/1.1 401 Unauthorized
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}
```

**4.3 — Rate limit (vòng lặp 12 request với cùng token, tài khoản `student` giới hạn 10 req/phút):**
```text
req 1..10: HTTP/1.1 200 OK
req 11: HTTP/1.1 429 Too Many Requests
req 12: HTTP/1.1 429 Too Many Requests

HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
retry-after: 58
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":58}}
```

### Exercise 4.4: Cost guard implementation
**Bản mẫu Part 4** (`04-api-gateway/production/cost_guard.py`): mỗi user có `UsageRecord` theo ngày (input/output tokens, số request). `check_budget(user_id)` chạy **trước** khi gọi LLM: tổng chi phí toàn hệ thống ≥ `global_daily_budget_usd` ($10) → 503; chi phí của user ≥ `daily_budget_usd` ($1) → **402 Payment Required**; gần ngưỡng thì log cảnh báo. Sau khi gọi LLM, `record_usage` cộng token và tính tiền theo giá input/output. Sau 10 request ở trên, `/me/usage` báo `cost_usd: 0.000189`, còn `budget_remaining_usd: 0.999811`. Hạn chế: state nằm trong RAM, restart là mất, và mỗi instance tự đếm riêng.

**Cách mình làm ở Part 6** (`06-lab-complete/app/cost_guard.py`):
- Cộng dồn chi phí vào **Redis** bằng `INCRBYFLOAT` với hai key `cost:daily:YYYY-MM-DD` và `cost:monthly:YYYY-MM` (UTC), có TTL (2 ngày / 32 ngày) nên tự hết hạn khi sang kỳ mới.
- `check_budget()` đọc hai key bằng `MGET` trước khi gọi LLM; vượt `DAILY_BUDGET_USD` hoặc `MONTHLY_BUDGET_USD` (đọc từ env) → **402**.
- `record_cost()` gọi sau khi LLM trả lời, tính `tokens/1000 × giá` rồi cộng vào cả hai key trong một transaction (`MULTI/EXEC`).
- State trong Redis nên mọi worker/instance dùng chung một ngân sách và còn nguyên sau restart.

**Vì sao cần cả rate limit lẫn cost guard:** rate limit chặn **tốc độ** (chống spam, DoS, một client chiếm hết tài nguyên), cost guard chặn **tổng tiền**. Một client gọi đúng 10 req/phút nhưng liên tục 24/7, hoặc gửi prompt rất dài, vẫn qua được rate limit mà đốt hết ngân sách. Chỉ cost guard mới chặn được trường hợp đó.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
_TODO_

## Part 6: Final Project

### Output `check_production_ready.py` (📋 Checklist Deliverable)
```text
_TODO: dán output của lần chạy đầy đủ trong WSL_
```

## AI log

| # | Phần | Hỏi gì | Nhận gì | Tự sửa / tự kiểm tra thế nào |
|---|------|--------|---------|------------------------------|
| 1 | Part 6 | Nhờ Claude Code (Opus 5.5) đọc `check_production_ready.py` và `06-lab-complete/README.md`, rồi hoàn thành các TODO trong `06-lab-complete/` | Code cho `app/config.py`, `auth.py`, `redis_client.py`, `rate_limiter.py` (sorted set + MULTI/EXEC), `cost_guard.py` (INCRBYFLOAT theo ngày/tháng), `main.py` (log JSON, `/health`, `/ready`, lifespan, exception handler); `Dockerfile` multi-stage, `.dockerignore`, `docker-compose.yml`, `.env.example`, `railway.toml`, `render.yaml` | _TODO: ghi bạn đã đọc hiểu / sửa gì_. Đã kiểm: script chấm trên Windows đạt 11/12 tiêu chí (42/42 check), image 63 MB, `docker stop` exit 0 trong 1.5s; bài test SIGTERM chạy trong container Linux trả exit 0; `docker compose up` gọi `/ask` có key trả 200. Lỗi tìm thấy và đã sửa: compose chưa truyền `AGENT_API_KEY` vào container |
| 2 | Part 7 Bước 1 | Nhờ Claude Code chạy load test, eval và sửa `KNOWLEDGE_BASE` theo CP7, rồi điền mục 1–3 của `BENCHMARK_REPORT.md` | Claude chạy các lệnh và lưu 10 file JSON trong `develop/reports/`. Claude thêm 2 entry KB (JWT, Kubernetes HPA) vào `develop/` và `production/utils/mock_llm.py`, rồi viết phần nhận xét và job CI mẫu | _TODO: ghi bạn đã đọc lại / sửa gì_. Mọi số trong báo cáo được trích bằng script từ file JSON, không gõ tay. Eval cuối báo `Quality gate ... ✅ PASS` (23/23, 0 regression). Gặp lỗi `UnicodeEncodeError` khi ghi JSON trên Windows (cp1252), sửa bằng `PYTHONUTF8=1` |
| 3 | Part 7 Bước 2 | Nhờ Claude Code chạy stack observability, Locust headless (scale 1/3), các thí nghiệm A–E, lấy trace từ Jaeger, rồi điền mục 4.0, 4.2, 4.3, 5 của `BENCHMARK_REPORT.md` | Claude sửa `production/Dockerfile`: thêm `PYTHONUSERBASE` vì user `agent` có home `/app`, nên container lỗi `No module named 'uvicorn'`. Claude viết script đo thời gian alert firing qua API Prometheus, tạo 2 file CSV của Locust, tìm trace chậm/lỗi qua Jaeger API v3 và grep log theo `trace_id`, rồi viết phần phân tích | _TODO: ghi bạn đã đọc lại / sửa gì_. Số trong bảng 5 lấy từ CSV, không lấy từ output console (console in cột p90 cạnh p95, suýt đọc nhầm). `promtool test rules` báo SUCCESS. Phát hiện `docker compose run locust` tạo lại agent với cấu hình mặc định |
