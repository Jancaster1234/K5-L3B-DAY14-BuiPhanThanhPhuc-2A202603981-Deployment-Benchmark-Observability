# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. _TODO_

### Exercise 1.2: Kết quả gọi bản develop
_TODO_

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config  | _TODO_  | _TODO_     | _TODO_         |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: _TODO_

### Exercise 2.3: Image size comparison
- Develop: _TODO_ MB
- Production: _TODO_ MB
- Difference: _TODO_ %

### Exercise 2.4: Docker Compose stack
_TODO_

## Part 3: Cloud Deployment

### Exercise 3.1 / 3.2: Deploy agent Part 6
- URL: _TODO_
- Screenshot: `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png`

### So sánh `railway.toml` và `render.yaml`
_TODO_

### Exercise 3.3: Pipeline Cloud Run
_TODO_

## Part 4: API Security

### Exercise 4.1-4.3: Test results
_TODO_

### Exercise 4.4: Cost guard implementation
_TODO_

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
