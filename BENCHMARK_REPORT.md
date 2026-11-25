# Benchmark & Observability Report — Day 14

**Họ tên:** Đinh Trần Nguyên (MSSV: 2A202603517)
**Ngày chạy:** 25/11/2026
**Môi trường:** Windows + WSL2 Ubuntu, Docker Desktop

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | Đảm bảo 95% người dùng nhận phản hồi dưới nửa giây, chấp nhận được cho tác vụ hội thoại. |
| Error rate `/ask` | < 2 % | Cho phép error budget 2% để chịu được lỗi upstream tạm thời từ LLM mà không ảnh hưởng đa số user. |
| Accuracy trên golden dataset | ≥ 85 % (mục tiêu 100%) | Đảm bảo agent trả lời đúng kiến thức cốt lõi và không bị regression khi cập nhật. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.79 | 189.3 | 218.9 | 1668.6 | 1.67% | PASS |
| 10 | 42.82 | 193.4 | 223.0 | 1680.7 | 0.33% | PASS |
| 50 | 104.41 | 206.1 | 278.2 | 1716.9 | 1.00% | PASS |
| 100 | 60.91 | 343.0 | 2836.9 | 3193.0 | 0.67% | FAIL |

**Nhận xét:** RPS không tăng tuyến tính mãi theo concurrency. Từ 1 lên 10 và 50, RPS tăng mạnh (4.79 -> 42.82 -> 104.41) vì agent xử lý bất đồng bộ (async I/O). Nhưng tới concurrency 100, server quá tải hàng đợi nên RPS tụt xuống 60.91 và p95 vọt lên 2836.9 ms (vi phạm SLO < 500 ms).

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 193.8 | 226.9 | 250.5 |
| 0.03 (mặc định) | 264.1 | 257.4 | 1706.9 |
| 0.1 | 345.2 | 1687.1 | 1718.9 |

**Nhận xét:** p99 (và p95 khi slow rate đạt 10%) nhạy nhất với request chậm. Không nên đặt SLO theo average vì khi có 3% request chậm ~1.7s, average chỉ tăng nhẹ (193.8 -> 264.1 ms) trong khi p99 đã vọt lên 1706.9 ms, che giấu trải nghiệm tệ của nhóm người dùng gặp tail latency.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 56.5 | 109.1 | 111.7 |
| Tổng cost (USD) | $0.000396 | $0.000762 | $0.000773 |
| Cost / câu đúng (USD) | $0.000396 | $0.00003629 | $0.00003361 |
| Quality gate | FAIL | PASS | PASS |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry so sánh JWT và API key vào `KNOWLEDGE_BASE` trong `utils/mock_llm.py` |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry về Kubernetes HPA (CPU, memory, custom metrics) vào `KNOWLEDGE_BASE` |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**
Agent v2 dùng retrieval nên tốn thêm ~20 ms latency và gấp đôi tổng token/chi phí ($0.000773 so với $0.000396) do phải đưa context vào prompt. Tuy nhiên accuracy tăng từ 4.3% lên 100%, giúp chỉ số cost / câu đúng giảm hơn 11 lần ($0.00003361 so với $0.000396). Để chặn merge khi chất lượng giảm, có thể dùng exit code của `run_eval.py` (trả về 1 nếu accuracy < `--min-accuracy` hoặc có regression) trong job CI/CD.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải (`screenshots/grafana-dashboard.png`)
- [x] Một trace chậm trong Jaeger (`screenshots/jaeger-slow-trace.png`)
- [x] Một trace lỗi trong Jaeger (`screenshots/jaeger-error-trace.png`)
- [x] Trang Alerts của Prometheus với ≥ 2 alert đang **firing** (`screenshots/prometheus-alert-firing.png`)

### 4.2 Thí nghiệm kích hoạt Alert (Fault Injection)

Đã chạy kết hợp `MOCK_ERROR_RATE=0.3` và `MOCK_SLOW_RATE=0.5` với tải 30 users trong 3 phút (`2459` requests `/ask`, `736` lỗi `503` = `29.93%`, `p95 = 1700 ms`):

| Thay đổi | Alert nào firing | Thời gian từ lúc đổi -> firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|
| `MOCK_ERROR_RATE=0.3` | `AgentHighErrorRate` (> 5%, `for: 1m`) | ~1 phút 15 giây | Error Rate (5xx) trên Grafana |
| `MOCK_SLOW_RATE=0.5` | `AgentHighLatencyP95` (> 1s, `for: 2m`) | ~2 phút 15 giây | Latency p50/p95/p99 trên Grafana |

**Nếu chất lượng trả lời bị giảm (như `AGENT_VERSION=v1`) nhưng không làm alert HTTP nào firing, phát hiện bằng cách nào?**
Chạy `run_eval.py` định kỳ hoặc trong CI/CD trước khi deploy (quality gate) trên golden dataset; trên production có thể kết hợp theo dõi feedback người dùng (thumbs up/down), LLM-as-a-judge trên mẫu traffic, và metric độ dài câu trả lời / tỉ lệ fallback.

### 4.3 Debug bằng trace

1. Trace chậm trên Jaeger (`screenshots/jaeger-slow-trace.png`, trace `a540e77`):
   - Tổng thời gian: 2.7s (2700 ms), gồm 4 spans.
   - `agent.retrieve_context`: 15.1 ms.
   - Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (~2.68s, ~99% tổng thời gian).
2. Trace lỗi và dòng log JSON tương ứng (grep theo `trace_id` `af7784b41711909a52667d768ae70c3e`):

```json
{"ts": "2026-11-25T14:58:04.642476+00:00", "level": "INFO", "service": "observable-agent", "instance": "04b7f032c59b", "event": "llm_error", "trace_id": "af7784b41711909a52667d768ae70c3e", "span_id": "48b475e80514dd12", "request_id": "2aa49774ef01", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút, endpoint `POST /ask`)

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 57.75 | 200 | 270 | 1700 | 67 / 7149 (0.94%) |
| 3 | 57.54 | 200 | 330 | 1700 | 63 / 7082 (0.89%) |

**Giải thích kết quả:** Scale từ 1 lên 3 instance trên cùng một máy không làm tăng RPS hay giảm p95 (p95 tăng nhẹ từ 270 ms lên 330 ms). Nút thắt nằm ở hai chỗ: (1) thời gian xử lý chủ yếu là độ trễ mô phỏng của LLM (~150–200 ms, và 3% tail latency ~1.7s) mà 1 instance async đã xử lý đủ 100 users (~58 RPS), và (2) cả 3 container agent, Nginx, Prometheus, Grafana, Jaeger và Locust đều chạy chung trên CPU máy local nên tăng thêm replica làm tăng tranh chấp CPU.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. Phải dùng p95/p99 để đặt SLO thay vì average để không bỏ sót tail latency.
  2. Eval trên golden dataset + quality gate là bắt buộc để đo chất lượng và ngăn regression trước khi deploy.
  3. Kết hợp Metrics (Grafana) + Traces (Jaeger) + Logs JSON có `trace_id` giúp đi từ triệu chứng tổng quan tới đúng span và nguyên nhân gốc.
- Nếu đưa agent lên production thật, sẽ thêm alert cho tỷ lệ timeout/rate-limit từ LLM provider, cảnh báo sắp chạm ngưỡng `DAILY_BUDGET_USD`, và alert khi kết nối Redis (`/ready`) mất ổn định.
