# Benchmark & Observability Report — Day 14

**Họ tên:** Vu Van Truong — 2A202603910 (K5-L3B)
**Ngày chạy:** 2026-11-26
**Môi trường:** Laptop AMD Ryzen 7 5800H (16 luồng), 15 GB RAM, Windows + WSL2 (Linux 6.18), Docker 29.9, Python 3.12. Load generator và agent chạy chung một máy.

Số liệu lấy từ: `reports/*.json` (load test, eval), `benchmark-results/scale_{1,3}_stats.csv` (Locust), ảnh trong `screenshots/`.

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | LLM call bình thường ~150–220 ms; 500 ms cho dư ~2× để chịu jitter + retrieval, vẫn dưới ngưỡng người dùng cảm nhận "chậm" (~1 s). Đặt theo p95 chứ không theo avg (xem 2.2). |
| Error rate `/ask` | < 2 % | Upstream mock có ~1 % lỗi nền; 2 % = error budget gấp đôi mức nền, alert ở 5 % (lỏng hơn SLO) để tránh alert fatigue. |
| Accuracy trên golden dataset | ≥ 80 % (gate CI); mục tiêu release = 100 % và 0 regression | 80 % là ngưỡng tối thiểu của `run_eval.py`; sau khi sửa KB, v2 đạt 23/23 nên khoá baseline ở 100 % để mọi thay đổi prompt/KB không được làm tụt câu nào. |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2, 300 request mỗi mức)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.75 | 188.1 | 216.4 | 1662.5 | 1.33 % | ✅ PASS |
| 10 | 40.43 | 186.9 | 217.3 | 1676.9 | 0.67 % | ✅ PASS |
| 50 | 118.31 | 194.9 | 254.0 | 1695.2 | 0.33 % | ✅ PASS |
| 100 | 61.09 | 355.8 | 3022.6 | 3294.5 | 0.67 % | ❌ FAIL (p95) |

**Nhận xét:** RPS tăng gần tuyến tính từ 1 → 10 (×8.5) vì mỗi request chủ yếu là *chờ* LLM (`asyncio.sleep`), một process async phục vụ được nhiều request chờ song song; theo Little's law RPS ≈ concurrency / latency ≈ 10 / 0.2 s ≈ 50 (đo được 40). Từ 10 → 50 RPS chỉ tăng ×2.9 và p95 bắt đầu nhích (217 → 254 ms): event loop của **một** process uvicorn bắt đầu bận (parse JSON, log, metrics, `random`), cộng thêm client load test chạy cùng máy tranh CPU. Ở 100, agent bị bão hoà: request xếp hàng trong event loop, p50 gần gấp đôi, p95 nhảy lên 3 s, RPS còn **giảm** xuống 61 (thời gian xếp hàng ăn hết lợi ích của concurrency). Điểm "gãy" nằm giữa 50 và 100 → muốn chịu tải cao hơn phải scale ngang (nhiều container) chứ không phải tăng concurrency trên một process.

### 2.2 Tail latency (concurrency 20, 300 request)

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 190.9 | 217.6 | 224.2 |
| 0.03 (mặc định) | 226.0 | 219.4 | 1701.3 |
| 0.1 | 326.6 | 1682.2 | 1706.4 |

**Nhận xét:** p99 nhạy nhất: chỉ 3 % request chậm đã đẩy p99 từ 224 → 1701 ms (×7.6) trong khi avg chỉ tăng 18 % và p95 gần như không đổi. Khi tỉ lệ chậm vượt 5 % (0.1) thì đến lượt p95 nhảy lên ~1.7 s. Average che giấu tail: avg = 226 ms trông "ổn" nhưng cứ 100 người thì vài người chờ 1.7 s. SLO đặt theo average sẽ báo xanh trong khi một nhóm người dùng có trải nghiệm rất tệ; percentile (p95/p99) mô tả đúng trải nghiệm của phần đuôi.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3 % (1/23) | 91.3 % (21/23) | **100 % (23/23)** |
| Avg tokens / câu | 56.7 | 109.1 | 116.7 |
| Tổng cost (USD) | 0.000399 | 0.000762 | 0.000817 |
| Cost / câu đúng (USD) | 0.00039855 | 0.00003627 | 0.00003550 |
| Quality gate | ❌ FAIL (exit 1) | ✅ PASS (exit 0) | ✅ PASS (exit 0, `--min-accuracy 1.0`, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách sửa:**

| ID | Keyword bị thiếu | Đã sửa gì |
|---|---|---|
| `security-02` "JWT khác gì API key?" | `token`, `hết hạn`, `chữ ký` | Thêm entry `(["jwt", "json web token"], …)` vào `KNOWLEDGE_BASE`: JWT là token có chữ ký số, có thời gian hết hạn (`exp`); API key là chuỗi cố định không tự hết hạn. |
| `scaling-03` "Kubernetes HPA scale dựa trên chỉ số nào?" | `cpu`, `memory`, `custom metrics` | Thêm entry `(["hpa", "horizontal pod autoscaler", "autoscal"], …)`: HPA scale theo CPU utilization, memory hoặc custom metrics. |

Hai entry thêm ở **cuối** danh sách và trigger keyword chỉ xuất hiện trong đúng 2 câu này (đã `grep` golden dataset), nên không đổi kết quả retrieval của 21 câu còn lại → 0 regression. Bản sửa được chép sang `production/utils/mock_llm.py` để stack Docker cũng đạt 100 %.

**Quality gate trong CI (Exercise 7.3):** chạy v1 với `--baseline reports/eval_v2.json` → accuracy 4.3 %, **20 câu regression**, **exit code = 1**. Job [`.github/workflows/eval-quality-gate.yml`](.github/workflows/eval-quality-gate.yml) bật agent v2 rồi chạy `run_eval.py --min-accuracy 1.0 --baseline reports/eval_v2_fixed.json`; exit code ≠ 0 làm step fail → job fail → đặt job này là *required status check* trong branch protection thì GitHub khoá nút Merge của PR.

**Trade-off v1 vs v2:** v2 (RAG) tốn gần **gấp đôi token** mỗi câu (+92 %), tổng cost +91 % và p95 latency +27 ms (bước retrieval ~20 ms), nhưng accuracy từ 4 % lên 91–100 %. Tính theo **cost / câu đúng** thì v2 rẻ hơn v1 khoảng **11 lần** (0.0000355 so với 0.000399 USD). Nghĩa là trả thêm chi phí và chút latency là đáng: chỉ số cần tối ưu là chi phí cho mỗi câu trả lời *đúng*, không phải chi phí mỗi request.

---

## 4. Observability stack (production)

Stack chạy bằng `docker compose -p k5-day14-observability …` (agent + Nginx + Prometheus + Grafana + Jaeger + Locust). Unit test alert: `promtool test rules alert_rules_test.yml` → `SUCCESS`.

> Sửa một lỗi trong `production/Dockerfile` của đề: user `agent` có HOME `/app` nên Python không thấy packages copy vào `/home/agent/.local` (`ModuleNotFoundError: uvicorn`, container restart liên tục). Thêm `ENV PYTHONUSERBASE=/home/agent/.local`.

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải — [`screenshots/grafana-dashboard.png`](screenshots/grafana-dashboard.png) (trong thí nghiệm A+B: Traffic 15.2 req/s, Error rate 20.29 % đỏ, p95 1.91 s đỏ, 2 instance, cost 1h $0.68)
- [x] Một trace chậm trong Jaeger — [`screenshots/jaeger-slow-trace.png`](screenshots/jaeger-slow-trace.png)
- [x] Một trace lỗi trong Jaeger — [`screenshots/jaeger-error-trace.png`](screenshots/jaeger-error-trace.png) (span `chat mock-llm-small` đỏ, HTTP 503, `error.type=upstream_unavailable`, `Events (1)` = event `exception`)
- [x] Trang Alerts của Prometheus — [`screenshots/prometheus-alert-firing.png`](screenshots/prometheus-alert-firing.png): **`AgentHighErrorRate` và `AgentHighLatencyP95` cùng FIRING**

### 4.2 Thí nghiệm "phá hệ thống" (Locust 30 user chạy nền suốt thí nghiệm)

Thời gian lấy từ một script poll `GET /api/v1/alerts` mỗi 5 s, ghi mọi lần đổi trạng thái.

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` (15:06:35) | `AgentHighErrorRate` (pending 15:07:02 → firing 15:08:02) | **1 phút 27 giây** | *Error rate* (20.29 %, đỏ) và *Requests/s theo status code* (đường 503 tăng) |
| B | `MOCK_SLOW_RATE=0.3` (cùng lúc với A, 15:06:35) | `AgentHighLatencyP95` (pending 15:06:52 → firing 15:08:53) | **2 phút 18 giây** | *Latency /ask p50/p95/p99* (p95 vọt lên ~1.9 s, p50 gần như không đổi) và *LLM latency p95* |
| C | `LLM_MODEL=mock-llm-large` (15:13:35) | `LLMCostBurnRateHigh` (pending 15:14:14 → firing 15:16:15) | **2 phút 40 giây** | *Cost burn rate (USD/giờ)*: đo được **$16.78/giờ** (ngưỡng $2) |
| D | `docker pause k5-day14-observability-agent-2` (15:13:54) | `AgentInstanceDown` (pending 15:14:04 → firing 15:14:34) | **40 giây** | *Instances up* giảm 2 → 1; *Requests/s theo instance* mất 1 đường. Locust thấy request treo tới 48.7 s (timeout ở Nginx) nhưng panel Error rate của agent **không** đỏ vì request đó chưa bao giờ tới agent. |
| E | `AGENT_VERSION=v1` (15:16:20) | Không có alert mới | — | *Phiên bản đang chạy* đổi sang `v1`; *Retrieval hit ratio* mất; *Tokens/s* giảm (input ~1000/s). Mọi panel hạ tầng vẫn xanh. |

Ghi chú: `LLMCostBurnRateHigh` cũng đã firing một lúc trước thí nghiệm do 2 lần Locust 100 user ở mục 5 (cost/giờ tỉ lệ với traffic, đo được $6.01/giờ); nó tự resolve lúc 15:10 trước khi chạy C, nên thời gian của C ở trên là từ trạng thái inactive.

**Thí nghiệm E không làm alert nào firing. Phát hiện loại sự cố này trong production bằng cách nào?**
Lỗi chất lượng không hiện trên metric hạ tầng (latency, error, saturation đều tốt hơn vì v1 rẻ và nhanh hơn). Cần: (1) **eval gate trong CI/CD** — `run_eval.py` với baseline chặn ngay từ lúc deploy, như job ở mục 3; (2) **online eval / canary**: chạy golden dataset định kỳ (synthetic check mỗi vài phút) vào production và đẩy `eval_accuracy` thành metric Prometheus có alert; (3) **metric proxy chất lượng** như retrieval hit ratio, tỉ lệ câu trả lời "generic/không biết", độ dài câu trả lời — alert khi hit ratio rơi về 0; (4) **tín hiệu người dùng**: thumbs-down rate, tỉ lệ hỏi lại; (5) alert khi `agent_info{version}` đổi ngoài quy trình release.

### 4.3 Debug bằng trace

Trace chậm nhất tìm được (qua Jaeger API v3, 994 trace `POST /ask` trong 15 phút):

- `trace_id`: `27a9219c7bebac6e77c2ba3dcef959af`
- Tổng thời gian: **1724.8 ms**
- Span chiếm nhiều thời gian nhất: **`chat mock-llm-small`** (**1695.6 ms, 98.3 %**). `agent.retrieve_context` chỉ 26.6 ms (1.5 %). → Request chậm do upstream LLM (request "slow" của mock), không phải code của agent hay bước retrieval.
- Dòng log tương ứng (`docker compose -p k5-day14-observability logs agent | grep 27a9219c…`):

```json
{"ts": "2026-11-26T08:06:54.079602+00:00", "level": "INFO", "service": "observable-agent", "instance": "db29da63ed6b", "event": "llm_call", "trace_id": "27a9219c7bebac6e77c2ba3dcef959af", "span_id": "7c9a2785bee73d46", "request_id": "9506d2504d10", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 76, "output_tokens": 39, "cost_usd": 3.5e-05, "llm_ms": 1694.7}
{"ts": "2026-11-26T08:06:54.079968+00:00", "level": "INFO", "service": "observable-agent", "instance": "db29da63ed6b", "event": "http_request", "trace_id": "27a9219c7bebac6e77c2ba3dcef959af", "span_id": "7c9a2785bee73d46", "request_id": "9506d2504d10", "method": "POST", "route": "/ask", "status": 200, "ms": 1724.2}
```

Trace lỗi: `trace_id` `197a09ccd194814aed28d65e0b497604`, 848.6 ms, HTTP 503; span `chat mock-llm-small` có status ERROR `LLMError: Upstream LLM error: 503 Service Unavailable (mock)`. Log cùng `trace_id`:

```json
{"ts": "2026-11-26T08:07:08.606097+00:00", "level": "INFO", "service": "observable-agent", "instance": "db29da63ed6b", "event": "llm_error", "trace_id": "197a09ccd194814aed28d65e0b497604", "span_id": "487a46a91f109e1d", "request_id": "b64fc240765b", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
{"ts": "2026-11-26T08:07:08.606348+00:00", "level": "INFO", "service": "observable-agent", "instance": "db29da63ed6b", "event": "http_request", "trace_id": "197a09ccd194814aed28d65e0b497604", "span_id": "487a46a91f109e1d", "request_id": "b64fc240765b", "method": "POST", "route": "/ask", "status": 503, "ms": 848.1}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, ramp 10/s, 2 phút)

Số liệu dòng `Aggregated` trong `benchmark-results/scale_{1,3}_stats.csv` (gồm `POST /ask` + `GET /health`):

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 66.21 | 190 | 220 | 1700 | 87 / 7887 (1.10 %) |
| 3 | 65.88 | 190 | 220 | 1700 | 65 / 7846 (0.83 %) |

**Giải thích kết quả:** Scale 1 → 3 **không** cải thiện RPS hay p95, vì ở tải này agent chưa phải nút thắt:

1. **Load generator là vòng kín**: mỗi user Locust chờ `between(0.5, 2)` s (trung bình 1.25 s) giữa các request, nên 100 user chỉ sinh tối đa ≈ 100 / (1.25 + 0.21) ≈ **68 req/s**, đúng con số đo được ở cả hai lần. Thêm instance không làm user gửi nhanh hơn.
2. **Latency bị chặn dưới bởi LLM**: p50 ~190 ms gần như bằng latency mock LLM (120 ms + jitter 0–60 ms + retrieval ~20 ms); p99 1700 ms = request "slow" 3 % (+1500 ms). Đây là thời gian *chờ upstream*, một instance async đã chờ song song được hàng chục request, nên chia tải ra 3 instance không rút ngắn được nó (panel *LLM latency p95* phẳng ở cả hai lần).
3. Một instance (0.5 CPU) ở ~66 req/s vẫn chưa bão hoà (in-flight thấp), nên thêm instance chỉ làm mỗi instance nhàn hơn. Lợi ích của 3 instance ở đây là **độ sẵn sàng** (mất 1 instance như thí nghiệm D vẫn còn phục vụ) chứ không phải latency. Muốn thấy scaling có tác dụng phải tăng tải tới mức 1 instance bão hoà (nhiều user hơn hoặc `wait_time` nhỏ hơn), như mức concurrency 100 ở mục 2.1 nơi một process đã gãy.

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. **Average nói dối, percentile thì không**: 3 % request chậm làm p99 tăng ×7.6 trong khi avg chỉ +18 %. SLO và alert phải đặt trên p95/p99.
  2. **Metric hạ tầng không bắt được lỗi chất lượng**: đổi sang v1 làm accuracy rơi từ 100 % xuống 4 % nhưng dashboard toàn xanh (thí nghiệm E). Eval trên golden dataset với quality gate trong CI là lớp bảo vệ riêng, bắt buộc phải có. Chỉ số để so sánh phiên bản là *cost / câu đúng* chứ không phải cost / request.
  3. **Ba trụ cột phải nối với nhau**: dashboard báo p95 đỏ → Jaeger chỉ ra 98 % thời gian nằm ở span LLM → `trace_id` grep ra đúng dòng log JSON. Và metric chỉ phản ánh những gì *chính service đó* thấy: lỗi 504 / request treo ở Nginx khi pause instance không hề hiện trên Error rate của agent.
- Nếu đưa agent này lên production thật, sẽ thêm:
  - Metric/alert **ở tầng load balancer** (Nginx 5xx/504, upstream response time) và **synthetic check từ bên ngoài** gọi `/ask` mỗi phút.
  - **Online eval**: chạy golden dataset định kỳ, xuất `eval_accuracy` thành gauge, alert khi < 95 %; alert khi *retrieval hit ratio* < 80 %.
  - Alert **cost theo ngày so với budget** (dự báo cháy budget, kiểu `predict_linear`) và cost theo model/version; alert tỉ lệ 402/429 (cost guard và rate limit của Part 6).
  - Alert **saturation**: `agent_in_flight_requests` cao kéo dài, kèm autoscaling theo in-flight hoặc RPS mỗi instance.
  - Alert **error budget burn rate** nhiều cửa sổ (1h/6h) thay cho ngưỡng cố định, và tail-based sampling cho trace (giữ 100 % trace lỗi/chậm, chỉ lấy mẫu trace bình thường).
