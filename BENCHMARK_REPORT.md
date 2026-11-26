# Benchmark & Observability Report — Day 14

> Mọi con số lấy từ report JSON trong `reports/` và CSV trong `benchmark-results/`.

**Họ tên:** Bui Phan Thanh Phuc
**Ngày chạy:** 2026-11-26
**Môi trường:** Laptop AMD Ryzen 7 PRO 4750U (8 nhân / 16 luồng), 31 GB RAM, Windows 10. Phần develop chạy bằng Python 3.12 trên Windows; agent và `load_test.py` chạy trên **cùng một máy**, nên tranh nhau CPU.

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Một request bình thường của mock mất khoảng 150–250 ms (latency 120 ms + jitter 0–60 ms + retrieval 20 ms + phần theo độ dài prompt). 500 ms chừa gấp khoảng 2 lần, đủ cho tải vừa phải nhưng vẫn bắt được khi hệ thống bắt đầu xếp hàng. |
| Error rate `/ask` | < 2 % | Mock có sẵn 1% lỗi upstream (`MOCK_ERROR_RATE=0.01`). Đặt 2% thì phần lỗi "bình thường" chỉ dùng hết một nửa error budget; vượt 2% nghĩa là có sự cố thật. |
| Accuracy trên golden dataset | ≥ 80 % (gate CI), không regression | 80% là mức sàn để được merge. Sau khi sửa KB, v2 đạt 100%, và lần chạy cuối dùng `--min-accuracy 1.0` + `--baseline` để giữ nguyên mức đó. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

300 request mỗi mức, file `reports/load_v2_c{1,10,50,100}.json`.

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 3.92 | 202.4 | 248.5 | 1699.4 | 1.00% | ✅ PASS |
| 10 | 34.08 | 198.8 | 467.1 | 1714.5 | 1.00% | ✅ PASS |
| 50 | 61.76 | 363.0 | 1534.9 | 2538.8 | 1.00% | ❌ FAIL (p95) |
| 100 | 38.52 | 1098.3 | 5436.0 | 5762.4 | 0.00% | ❌ FAIL (p95) |

**Nhận xét:** RPS **không** tăng tuyến tính theo concurrency.

- **Từ 1 lên 10:** RPS tăng 8.7 lần (3.92 → 34.08), gần tuyến tính, p50 vẫn ~200 ms. Lý do: mock LLM dùng `asyncio.sleep`, nên trong lúc "chờ LLM" event loop phục vụ được request khác.
- **Lên 50:** RPS chỉ tăng thêm 1.8 lần (61.76), p50 lên 363 ms và p95 vượt SLO.
- **Lên 100:** RPS **giảm** còn 38.52 và p50 lên 1098 ms. Hệ thống đã bão hoà. Agent develop chạy **một process uvicorn**: một event loop vừa parse JSON, vừa ghi log, vừa cập nhật metrics cho mọi request. Thêm vào đó, `load_test.py` chạy trên cùng máy, giành CPU với agent. Khi vượt mức bão hoà, request phải xếp hàng. Theo Little's law: thời gian trung bình ≈ concurrency / RPS = 100 / 38.5 ≈ 2.6 s. Ra thêm concurrency chỉ làm dài hàng đợi, không làm tăng throughput.
- **Kết luận:** với SLO p95 < 500 ms, một instance chịu được khoảng **10–20 request đồng thời**. Muốn hơn thì phải scale ngang (thêm worker hoặc instance).

### 2.2 Tail latency

300 request, concurrency 20, file `reports/load_v2_slow_{0,0.03,0.1}.json`.

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 217.7 | 468.5 | 490.6 |
| 0.03 (mặc định) | 226.9 | 489.8 | 578.9 |
| 0.1 | 375.3 | 1709.7 | 1740.7 |

**Nhận xét:**

- **p95 và p99 nhạy nhất với request chậm.** Đổi từ 0 lên 0.1, avg chỉ tăng 1.7 lần (217.7 → 375.3 ms), nhưng p95 tăng 3.6 lần (468.5 → 1709.7 ms). Với 10% request chậm thêm 1500 ms, hơn 5% số request thuộc nhóm chậm, nên p95 rơi hẳn vào nhóm đó.
- **Kết quả 0.03 dao động theo mẫu.** Ở lần đo này max là 2026 ms, tức là có request chậm, nhưng p99 chỉ 578.9 ms. Với 300 request, p99 chỉ dựa trên ~3 request chậm nhất, nên đo lại sẽ ra số khác. Muốn ước lượng p99 tin cậy cần nhiều request hơn. Cũng ở lần này, error rate là 2.67% (8/300 lỗi 503 từ mock), vượt SLO 2%: đây là ngẫu nhiên của mock lỗi 1%, không phải do slow rate.
- **Vì sao không đặt SLO theo average:** avg trộn 90% request nhanh với 10% request rất chậm thành một con số trông "ổn" (375 ms < 500 ms), trong khi cứ 10 người dùng thì 1 người chờ gần 2 giây. SLO theo p95/p99 đo đúng trải nghiệm của nhóm người dùng tệ nhất, đó là thứ người dùng cảm nhận và phàn nàn.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

File `reports/eval_v1.json`, `reports/eval_v2.json`, `reports/eval_v2_fixed.json`.

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100% (23/23) |
| Avg tokens / câu | 57.3 | 109.3 | 117.2 |
| Tổng cost (USD) | 0.000407 | 0.000764 | 0.000821 |
| Cost / câu đúng (USD) | 0.00040695 | 0.00003639 | 0.00003569 |
| Quality gate | FAIL | PASS (≥ 80%, so với v1: 20 câu cải thiện, 0 regression) | PASS (≥ 100%, so với v2: 2 câu cải thiện, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` (JWT khác gì API key) | `token`, `hết hạn`, `chữ ký` | Thêm entry KB kích hoạt bằng `jwt` / `json web token`: JWT là token có chữ ký số, có thời điểm hết hạn (`exp`); API key là chuỗi cố định, không tự hết hạn. |
| `scaling-03` (Kubernetes HPA) | `cpu`, `memory`, `custom metrics` | Thêm entry KB kích hoạt bằng `hpa` / `horizontal pod autoscaler` / `autoscal`: HPA scale số pod theo CPU, memory hoặc custom metrics. |

Cả hai câu sai vì retrieval không tìm được tài liệu nào (KB không có từ khoá khớp), nên v2 rơi về câu trả lời chung chung. Chỉ sửa `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` và chép sang `production/utils/mock_llm.py`, không đụng `golden_dataset.jsonl`.

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**

- **Chi phí mỗi câu:** v2 tốn gần gấp đôi token (57.3 → 109.3) vì prompt có thêm context từ retrieval, nên tổng cost tăng 1.9 lần.
- **Chi phí mỗi câu đúng:** v2 **rẻ hơn khoảng 11 lần** (0.000407 → 0.0000364 USD). v1 rẻ nhưng gần như toàn sai, nên tiền đó coi như bỏ đi. Đây là lý do so hai phiên bản bằng "cost / câu đúng" thay vì "cost / câu".
- **Latency:** v2 chậm hơn một chút (p50 172.1 → 204.9 ms) vì thêm bước retrieval và prompt dài hơn. p95 trong eval (1681.8 / 1553.2 / 236.4 ms) không dùng để so sánh được: chỉ có 23 mẫu, một request chậm của mock là đủ làm p95 nhảy vọt. Latency phải đo bằng load test.
- **Sửa KB:** thêm 7.9 token / câu và 0.000057 USD tổng, đổi lại +8.7 điểm % accuracy. Rất đáng.

**Quality gate trong CI (Exercise 7.3):** `run_eval.py` trả **exit code 1** khi accuracy < `--min-accuracy` hoặc có câu bị regression so với `--baseline`. Đã thử: chạy v1 với `--baseline reports/eval_v2.json` thì gate báo 20 câu regression và exit code = 1. Trong GitHub Actions, một step trả exit code ≠ 0 làm job fail. Nếu đặt job này là *required status check* trong branch protection thì PR không merge được:

```yaml
# .github/workflows/eval.yml
name: eval-quality-gate
on: [pull_request]
jobs:
  eval:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: 06-lab-complete/benchmark-observability/develop
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - name: Start agent v2
        run: |
          AGENT_VERSION=v2 PORT=8002 python app.py &
          for i in $(seq 1 30); do curl -sf localhost:8002/health && break; sleep 1; done
      - name: Quality gate (exit 1 → job fail → chặn merge)
        run: python eval/run_eval.py --url http://localhost:8002 --min-accuracy 1.0 --baseline reports/eval_v2_fixed.json
```

---

## 4. Observability stack (production)

### 4.0 Exercise 7.4 — đọc dashboard khi có tải

Đo bằng chính các câu PromQL của dashboard, cấu hình mặc định, 2 instance, Locust 30 users (ramp 5), sau ~3 phút có tải.

1. **p95 = 296.8 ms, avg = 247.9 ms** (p99 = 1859 ms). p95 chỉ cao hơn avg ~49 ms, vì ~97% request có latency tương tự nhau (một lần gọi LLM ~200 ms). Nhưng p99 cao gấp 7.5 lần avg: 3% request "slow" của mock (`MOCK_SLOW_RATE=0.03`) ít hơn 5% nên p95 không thấy, chỉ p99 lộ ra. Nhìn avg hoặc p95 sẽ không biết là có tail latency.
2. **Requests/s theo instance:** 9.06 và 9.12 req/s. Nginx chia gần như đều (round-robin) cho 2 instance.
3. **Input tokens/s (1419) lớn hơn output tokens/s (754)** gần gấp đôi. Prompt gửi vào LLM gồm system prompt + đoạn tài liệu lấy từ retrieval + câu hỏi, còn output chỉ là câu trả lời. RAG làm prompt dài ra, nên chi phí input là phần lớn.
4. **`curl http://localhost:8080/metrics` → `HTTP/1.1 404 Not Found`.** Nginx chặn `/metrics` từ bên ngoài. Prometheus scrape thẳng từng agent qua mạng nội bộ Docker (`agent:8000`), nên không cần đi qua Nginx. Không công khai `/metrics` vì metrics lộ thông tin nội bộ (route, model, chi phí, số instance), và ai cũng gọi được thì thành điểm tấn công.

### 4.1 Screenshot

- [ ] Dashboard Grafana khi đang có tải (thấy rõ Golden Signals)
- [ ] Một trace chậm trong Jaeger (thấy span nào chiếm thời gian)
- [ ] Một trace lỗi trong Jaeger (có event `exception`)
- [ ] Trang Alerts của Prometheus với ít nhất 1 alert đang **firing**

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
Điều kiện đo: 3 instance, Locust 10 users chạy nền suốt thí nghiệm. Thời gian tính từ lúc `docker compose up -d` (hoặc `docker pause`) trả về tới lúc rule chuyển `firing`, đo bằng script hỏi `GET /api/v1/rules` của Prometheus mỗi 5 giây (nên sai số ±5 s). Mỗi thí nghiệm đều trả cấu hình về mặc định và chờ alert về `inactive` trước khi làm thí nghiệm tiếp theo.

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` | 91 s (pending sau 32 s, rồi chờ `for: 1m`). Lúc firing error rate = 21.2%. Trả về mặc định: inactive sau 53 s | Error rate (Golden Signals) chuyển đỏ; req/s theo status thấy 5xx tăng |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` | 149 s (pending sau 27 s, rồi chờ `for: 2m`). Lúc firing p95 = 1.92 s trong khi avg = 0.65 s. Trả về: inactive sau 48 s | Latency p50/p95/p99/avg: p95/p99 vọt lên ~1.9 s, avg tăng ít hơn nhiều |
| C | `LLM_MODEL=mock-llm-large` | `LLMCostBurnRateHigh` (lúc đó `AgentHighLatencyP95` đang pending) | 175 s (pending sau 53 s). Cost/giờ tăng từ 0.58 → 6.53 USD và còn đang tăng, vì `rate(...[5m])` cần 5 phút mới phản ánh đủ; p95 lên 0.74 s. Trả về: inactive sau 265 s (cũng do cửa sổ 5 phút) | Cost USD/giờ và tokens/s (hàng LLM); LLM latency p95 |
| D | `docker pause` 1 instance | `AgentInstanceDown` | 42 s (pending sau 11 s, rồi chờ `for: 30s`). `up` của instance bị pause = 0, hai instance còn lại = 1. Error rate do agent tự đo vẫn chỉ 2.07% (xanh). Unpause: inactive sau 6 s | Instances up (Golden Signals) giảm từ 3 xuống 2; req/s theo instance mất một đường |
| E | `AGENT_VERSION=v1` | **Không có.** Kiểm tra mỗi phút trong 4 phút, cả 4 rule đều `inactive` | — | Retrieval hit ratio: hit 5.90/s → 0; tokens/s giảm từ 712 → 352; bảng version: cả 3 instance `version="v1"` |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

v1 vẫn nhanh, không lỗi, còn rẻ hơn, nên mọi metric hạ tầng đều xanh. Đây là sự cố **chất lượng**, không phải sự cố hạ tầng. Cách phát hiện:

1. **Chặn trước khi deploy:** chạy eval golden dataset trong CI với quality gate (`run_eval.py --baseline`, như mục 3). v1 so với baseline v2 cho exit code 1 (20 câu regression), nên bản này sẽ không bao giờ được merge.
2. **Synthetic eval trên production:** định kỳ (vd 5 phút một lần) gửi vài câu hỏi golden vào URL thật, xuất accuracy thành metric (vd `agent_eval_accuracy`), alert khi tụt dưới ngưỡng.
3. **Alert trên metric gián tiếp của chất lượng:** retrieval hit ratio giảm đột ngột (ở đây về 0), tokens/câu giảm mạnh, hoặc `agent_info{version}` đổi ngoài lịch deploy.
4. **Tín hiệu từ người dùng:** tỉ lệ thumbs-down / báo sai, hoặc LLM-as-judge chấm một mẫu nhỏ câu trả lời thật, rồi đưa thành metric và alert.

**Thêm 2 quan sát:**

- **Ngưỡng cost alert phụ thuộc lưu lượng.** Lúc chạy benchmark 100 users, `LLMCostBurnRateHigh` đã firing với model nhỏ (6.13 USD/giờ > 2). Ngưỡng tuyệt đối $2/giờ chỉ đúng với một mức traffic nhất định. Nên alert thêm theo **cost / request** để phân biệt "traffic tăng" với "mỗi request đắt lên" (như thí nghiệm C).
- **Khi chạy `docker compose run locust`, Compose khởi động lại các service phụ thuộc (nginx → agent) theo biến môi trường của chính lệnh đó.** Đổi `MOCK_*` bằng biến shell cho `up` rồi chạy `run` không kèm biến thì agent bị tạo lại với cấu hình mặc định. Phải bật tải trước rồi mới đổi cấu hình, hoặc ghi cấu hình vào file `.env`.

### 4.3 Debug bằng trace

**Trace trong ảnh `screenshots/jaeger-slow-trace.png`.** Đây là trace chậm nhất (không lỗi) trong 136 trace của 5 phút gần nhất; cấu hình `MOCK_SLOW_RATE=0.3`, 2 instance, 30 users.

- `trace_id`: `70d3c751d2eaa2d0c9db99709ec11d8a`
- Tổng thời gian: 1723.1 ms (span `POST /ask`)
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (1693.5 ms, 98.3%). `agent.retrieve_context` chỉ mất 24.5 ms (1.4%). Nút thắt nằm ở LLM, không phải ở code của agent.
- Dòng log tương ứng (`docker compose -p k5-day14-observability logs agent | grep 70d3c751...`). `llm_ms` = 1693.3 khớp span `chat`, `ms` = 1721.6 khớp span gốc:

```json
{"ts": "2026-11-26T16:06:35.561473+00:00", "level": "INFO", "service": "observable-agent", "instance": "9a1f3e128c87", "event": "llm_call", "trace_id": "70d3c751d2eaa2d0c9db99709ec11d8a", "span_id": "c55fe756537f9032", "request_id": "be6731d0483c", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 95, "output_tokens": 55, "cost_usd": 4.7e-05, "llm_ms": 1693.3}
{"ts": "2026-11-26T16:06:35.562401+00:00", "level": "INFO", "service": "observable-agent", "instance": "9a1f3e128c87", "event": "http_request", "trace_id": "70d3c751d2eaa2d0c9db99709ec11d8a", "span_id": "c55fe756537f9032", "request_id": "be6731d0483c", "method": "POST", "route": "/ask", "status": 200, "ms": 1721.6}
```

**Trace lỗi trong ảnh `screenshots/jaeger-error-trace.png`:** `trace_id` `3b6ddf0d1752038539ceb87093ebee4b`, tổng 103.4 ms. Span `chat mock-llm-small` (77.9 ms) và `agent.run` có status `LLMError: Upstream LLM error: 503 Service Unavailable (mock)` cùng event `exception`. Lỗi đến sớm vì mock trả lỗi sau một nửa latency. Log:

```json
{"ts": "2026-11-26T16:06:38.180100+00:00", "level": "INFO", "service": "observable-agent", "instance": "9a1f3e128c87", "event": "llm_error", "trace_id": "3b6ddf0d1752038539ceb87093ebee4b", "span_id": "778e609348b92134", "request_id": "af521f17c9f2", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
{"ts": "2026-11-26T16:06:38.181127+00:00", "level": "INFO", "service": "observable-agent", "instance": "9a1f3e128c87", "event": "http_request", "trace_id": "3b6ddf0d1752038539ceb87093ebee4b", "span_id": "778e609348b92134", "request_id": "af521f17c9f2", "method": "POST", "route": "/ask", "status": 503, "ms": 101.6}
```

**Trace phụ (lúc benchmark 1 instance, 100 users; Jaeger lưu trace trong RAM nên trace này không còn để chụp).** Đây là trace chậm nhất trong 379 trace ≥ 1.5 s lúc đó:

- `trace_id`: `754a55f1c0abb2a230c6891041c0b10f`
- Tổng thời gian: 2407.8 ms (span `POST /ask`)
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (1684.2 ms, 69.9%). Đây là request rơi vào nhóm "slow" của mock (`MOCK_SLOW_MS=1500`); `agent.retrieve_context` chỉ mất 19.6 ms (0.8%).
- **Khoảng trống đáng chú ý:** `agent.run` mất 1704.3 ms nhưng `POST /ask` mất 2407.8 ms. Khoảng 700 ms nằm ngoài code của agent: request phải chờ event loop của một instance bị giới hạn 0.5 CPU đang phục vụ 100 users.
- Dòng log tương ứng (`docker logs ... | grep 754a55f1...`). `llm_ms` = 1684.0 khớp với span `chat`, `ms` = 2405.7 khớp với span gốc:

```json
{"ts": "2026-11-26T08:06:16.883152+00:00", "level": "INFO", "service": "observable-agent", "instance": "d04138eb037e", "event": "llm_call", "trace_id": "754a55f1c0abb2a230c6891041c0b10f", "span_id": "352c04ab390ac4c8", "request_id": "a592479644b7", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 74, "output_tokens": 37, "cost_usd": 3.3e-05, "llm_ms": 1684.0}
{"ts": "2026-11-26T08:06:16.884131+00:00", "level": "INFO", "service": "observable-agent", "instance": "d04138eb037e", "event": "http_request", "trace_id": "754a55f1c0abb2a230c6891041c0b10f", "span_id": "352c04ab390ac4c8", "request_id": "a592479644b7", "method": "POST", "route": "/ask", "status": 200, "ms": 2405.7}
```

**Trace lỗi phụ (cùng đợt benchmark đó):** `trace_id` `447a2143626f4a09b4adcb2f70f6ee08`, tổng 141.4 ms. Span `chat mock-llm-small` (100.6 ms) có status `ERROR` (`LLMError: Upstream LLM error: 503 Service Unavailable (mock)`) và event `exception`. Lỗi đến sớm hơn một response bình thường vì mock trả lỗi sau một nửa latency. Dòng log:

```json
{"ts": "2026-11-26T08:04:51.217902+00:00", "level": "INFO", "service": "observable-agent", "instance": "d04138eb037e", "event": "llm_error", "trace_id": "447a2143626f4a09b4adcb2f70f6ee08", "span_id": "515f481a1bfb328c", "request_id": "3e556b6e8243", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
Số liệu dòng `/ask` trong `benchmark-results/scale_1_stats.csv` và `scale_3_stats.csv` (mỗi agent giới hạn 0.5 CPU / 256 MB).

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 57.16 | 210 | 790 | 1700 | 64 / 6840 (0.94%) |
| 3 | 57.78 | 210 | 670 | 1700 | 70 / 6917 (1.01%) |

**Giải thích kết quả:** Scale lên 3 instance **chỉ cải thiện p95 một chút** (790 → 670 ms). RPS, p50 và p99 gần như không đổi.

- **RPS không tăng vì giới hạn nằm ở phía tạo tải, không phải ở server.** Locust là tải "vòng kín": 100 user, mỗi user chờ trả lời rồi nghỉ 0.5–2 s (trung bình 1.25 s). Throughput tối đa ≈ 100 / (1.25 + ~0.3) ≈ 64 req/s (tính cả `/health`). Cả hai lần đo đều chạm đúng mức đó (Aggregated 62.80 và 63.64 req/s). Thêm instance không tạo thêm người dùng.
- **p95 giảm** vì với 1 instance (0.5 CPU), request phải xếp hàng chờ event loop. Trace chậm nhất ở mục 4.3 có ~700 ms nằm ngoài `agent.run`. Chia tải ra 3 instance thì hàng đợi ngắn lại.
- **Nút thắt thật là latency của LLM.** p50 = 210 ms là latency một lần gọi mock LLM; p99 = 1700 ms là các request "slow" của mock (+1500 ms), xuất hiện ở bất kỳ instance nào. Scale ngang không làm LLM nhanh hơn. Failures ~1% cũng đến từ `MOCK_ERROR_RATE=0.01` của upstream, không phải do quá tải.
- Muốn giảm p95/p99 phải xử lý phía LLM: timeout + retry/hedging cho request chậm, cache câu trả lời, hoặc dùng model/provider nhanh hơn.

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. ...
  2. ...
  3. ...
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?

...
