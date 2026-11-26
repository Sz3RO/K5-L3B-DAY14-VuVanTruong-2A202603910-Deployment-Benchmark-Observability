# Day 14 Lab — Mission Answers

> Môi trường chạy: WSL2 (`python -c "import os; print(os.name)"` → `posix`), Docker 29.9, Python 3.12 (venv `.venv/` ở gốc repo).
> Mọi output dưới đây là output thật của lệnh đã chạy ngày 2026-11-26. Token/API key đã được che.

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns trong `01-localhost-vs-production/develop/app.py`

1. **Hardcode secret** — dòng 17–18: `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`, `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Push lên GitHub là lộ key và mật khẩu DB ngay; đổi key phải sửa code và deploy lại.
2. **Log ra secret** — dòng 34: `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`. Key nằm trong log, ai đọc được log (log aggregator, CI) là có key.
3. **`print()` thay vì logging có cấu trúc** — dòng 33, 38: không có level, timestamp, không parse được bằng Loki/Datadog, còn log luôn cả nội dung câu hỏi/câu trả lời của người dùng.
4. **Config cố định trong code** — dòng 21–22: `DEBUG = True`, `MAX_TOKENS = 500`. Không đổi được giữa dev/staging/prod mà không sửa code (vi phạm 12-Factor "config in env").
5. **Không có health check** — không có `/health` hay `/ready` (gọi `/health` → **404**, xem output bên dưới). Platform không biết agent chết để restart, load balancer không biết instance chưa sẵn sàng.
6. **Bind `localhost`** — dòng 51: `host="localhost"` chỉ nhận kết nối từ chính máy đó; trong container/cloud không ai gọi vào được (phải là `0.0.0.0`).
7. **Port cố định** — dòng 52: `port=8000`; Railway/Render/Cloud Run cấp port qua biến `PORT`, app không đọc nên không nhận được traffic.
8. **`reload=True` trong production** — dòng 53: chế độ dev, theo dõi file và tự restart, tốn tài nguyên và không ổn định.
9. **Không xử lý shutdown** — không có lifespan/SIGTERM handler, không đóng kết nối; input nhận qua query string `question: str` không có validate độ dài.

### Exercise 1.2: Chạy bản develop

```text
$ curl -X POST "http://localhost:8000/ask?question=Hello"
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
$ curl -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP 422          # bản develop chỉ nhận question qua query
$ curl localhost:8000/health
HTTP 404          # không có health check
--- server stdout ---
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this     ← secret bị in ra log
[DEBUG] Response: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.
```

Chạy được, nhưng chưa production-ready: lộ secret trong log, không health check, không chạy được trong container.

Bản production (`cp .env.example .env && python app.py`):

```text
$ curl -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","model":"gpt-4o-mini"}
$ curl localhost:8000/health
{"status":"ok","uptime_seconds":0.6,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T08:18:54.070598+00:00"}
$ curl localhost:8000/ready
{"ready":true}
--- server log ---
WARNING:root:OPENAI_API_KEY not set — using mock LLM
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

### Exercise 1.3: Bảng so sánh

| Feature | Develop | Production | Tại sao quan trọng? |
|---|---|---|---|
| Config | Hardcode trong code (`DEBUG = True`, key, DB URL) | `config.py` đọc env (`os.getenv`), `.env.example`, `validate()` fail fast nếu thiếu `AGENT_API_KEY` ở production | Cùng một image chạy được ở dev/staging/prod; đổi config không cần build lại; secret không nằm trong git. |
| Secrets | `sk-...` và `password123` trong source, còn bị `print` ra log | Đọc từ env, không bao giờ log key; chỉ log `question_length` | Lộ key = mất tiền và dữ liệu; log thường được nhiều người/hệ thống đọc. |
| Health check | Không có (`/health` → 404) | `/health` (liveness, uptime, version) + `/ready` (readiness, 503 khi chưa sẵn sàng) | Platform dùng liveness để restart container treo; load balancer dùng readiness để không gửi traffic vào instance đang khởi động/tắt. |
| Logging | `print()` text tự do | JSON (`{"time","level","msg"}`, event `agent_request`/`agent_response`) | Parse/lọc/alert được trong log aggregator; có level; liên kết được với trace. |
| Shutdown | Đột ngột, không có hook | `lifespan` (startup/shutdown) + handler `SIGTERM`, `is_ready=False` khi tắt | Platform gửi SIGTERM khi deploy/scale down; graceful shutdown cho request đang chạy hoàn thành, không trả lỗi cho người dùng. |
| Host / Port | `localhost:8000` cố định, `reload=True` | `0.0.0.0`, `PORT` từ env, reload chỉ khi `DEBUG=true` | Container chỉ nhận traffic khi bind `0.0.0.0`; cloud cấp port động. |
| CORS / Input | Không có; question qua query | CORS theo `ALLOWED_ORIGINS`; body JSON, 422 khi thiếu `question` | Giới hạn origin được gọi API; validate input rõ ràng. |

---

## Part 2: Docker

### Exercise 2.1: Câu hỏi về `02-docker/develop/Dockerfile`

1. **Base image:** `python:3.11` — bản Python đầy đủ (Debian + build tools), rất nặng (~1 GB trước khi cài gì).
2. **Working directory:** `/app` (`WORKDIR /app`) — mọi lệnh `COPY`/`RUN`/`CMD` sau đó chạy tương đối với thư mục này.
3. **Vì sao `COPY requirements.txt` trước:** Docker cache theo layer. Requirements ít thay đổi, code thay đổi liên tục. Copy requirements → `pip install` trước rồi mới copy code thì khi chỉ sửa code, layer `pip install` vẫn dùng lại từ cache, build lại mất vài giây thay vì cài lại toàn bộ dependencies. Nếu `COPY . .` trước thì mọi thay đổi code đều làm mất cache của bước cài đặt.
4. **CMD vs ENTRYPOINT:** `ENTRYPOINT` là chương trình cố định của container; `CMD` là lệnh/tham số mặc định, bị ghi đè dễ dàng bằng `docker run image <lệnh khác>`. Dùng chung thì `CMD` trở thành tham số mặc định cho `ENTRYPOINT`. Dockerfile này dùng `CMD ["python", "app.py"]` (exec form → `python` là PID 1, nhận được SIGTERM).

### Exercise 2.2: Build và chạy (build từ gốc repo)

```text
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -p 8000:8000 my-agent:develop
$ curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker%3F"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
$ curl localhost:8000/health
{"status":"ok","uptime_seconds":0.5,"container":true}
```

### Exercise 2.3: Multi-stage build và so sánh kích thước

- **Stage 1 (`builder`, `python:3.11-slim`):** cài `gcc`, `libpq-dev` rồi `pip install --user -r requirements.txt` vào `/root/.local`. Stage này có compiler, apt cache… và **không** được deploy.
- **Stage 2 (`runtime`, `python:3.11-slim`):** tạo user non-root `appuser`, chỉ `COPY --from=builder /root/.local` (packages đã cài) + `main.py` + `utils/`, có `HEALTHCHECK`, `USER appuser`, chạy `uvicorn` 2 workers.
- **Vì sao nhỏ hơn:** base `slim` thay vì bản đầy đủ (~1 GB); compiler/header/apt cache nằm ở stage builder và bị bỏ lại; image cuối chỉ có Python runtime + site-packages + code.

```text
$ docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .
$ docker images my-agent
REPOSITORY   TAG        SIZE
my-agent     advanced   240MB
my-agent     develop    1.67GB
```

- Develop: **1667.8 MB** (`docker image inspect` = 1 667 779 995 byte)
- Production: **239.8 MB** (239 815 273 byte)
- Chênh lệch: **giảm 85.6 %** (production nhỏ hơn ~7 lần)

### Exercise 2.4: Docker Compose stack

```text
                    ┌───────────────────────────── network "internal" (bridge) ──────────────┐
  client ──:80/:443─▶  nginx (reverse proxy, LB,  ──▶  agent:8000 (FastAPI, multi-stage image) │
                    │  rate limit 10r/s, security        │            │                       │
                    │  headers)                          ▼            ▼                       │
                    │                               redis:6379    qdrant:6333                 │
                    │                               (session,     (vector DB                  │
                    │                                rate limit)   cho RAG)                   │
                    └──────────────────────────────────────────────────────────────────────────┘
```

- **4 service:** `agent` (build từ `02-docker/production/Dockerfile`, target `runtime`), `redis` (redis:7-alpine, healthcheck `redis-cli ping`), `qdrant` (v1.9.0, healthcheck TCP 6333), `nginx` (nginx:alpine, cổng 80/443 — **duy nhất** service mở ra ngoài).
- **Giao tiếp:** tất cả chung network `internal`, gọi nhau bằng tên service qua DNS của Docker (`redis://redis:6379/0`, `http://qdrant:6333`, Nginx `upstream agent_backend { server agent:8000; }`). Agent `depends_on` redis + qdrant với `condition: service_healthy` nên chỉ start khi 2 service kia healthy; agent không publish port, chỉ đi qua Nginx.

```text
$ touch .env.local && docker compose -p k5-day14-docker up -d --build
$ docker compose -p k5-day14-docker ps
SERVICE   IMAGE                   STATUS                           PORTS
agent     k5-day14-docker-agent   Up 1 second (health: starting)   8000/tcp
nginx     nginx:alpine            Up 1 second                      0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
qdrant    qdrant/qdrant:v1.9.0    Up 12 seconds (healthy)          6333-6334/tcp
redis     redis:7-alpine          Up 12 seconds (healthy)          6379/tcp
$ curl -i http://localhost/health
HTTP/1.1 200 OK
Server: nginx
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
{"status":"ok","uptime_seconds":0.4,"version":"2.0.0","timestamp":"2026-11-26T08:21:15.793275"}
$ curl -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question":"Explain microservices"}'
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

Header `Server: nginx` và các security header chứng minh request đi qua Nginx rồi mới tới agent.

---

## Part 3: Cloud Deployment

### Exercise 3.2 (nền tảng chính): Render — deploy agent Part 6

- **URL:** https://day14-production-agent-b6ln.onrender.com
- Deploy bằng **Render Blueprint** từ `06-lab-complete/render.yaml` (Blueprint Path `06-lab-complete/render.yaml`, branch `main`, auto-sync). Blueprint tạo web service Docker `day14-production-agent-b6ln` + Key Value (Redis) `day14-agent-redis-b6ln`, plan free, region Singapore. `AGENT_API_KEY` do Render sinh (`generateValue: true`), `REDIS_URL` lấy từ connection string của Redis (`fromService`). Chi tiết: [DEPLOYMENT.md](DEPLOYMENT.md).
- 4 lệnh tự kiểm CP6:

```text
$ curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     → 200
$ curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      → 200   (đã kết nối Redis trên Render)
$ curl ... -X POST "$BASE/ask" -d '{"question":"Hello"}'       → 401   (không có key; sai key cũng 401)
$ curl ... -X POST "$BASE/ask" -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'  → 200
$ python check_production_ready.py --public-url https://day14-production-agent-b6ln.onrender.com
     ✅ Public URL: GET https://day14-production-agent-b6ln.onrender.com/health → 200
```

- Screenshot: [screenshots/dashboard.png](screenshots/dashboard.png), [screenshots/running.png](screenshots/running.png), [screenshots/test.png](screenshots/test.png)

### Nền tảng còn lại: so sánh `railway.toml` với `render.yaml` (của `06-lab-complete/`)

| | `railway.toml` | `render.yaml` |
|---|---|---|
| Phạm vi | Cấu hình build/deploy **một service**; project, Redis, biến môi trường tạo bằng CLI/dashboard (`railway init`, `railway variables set`, `railway add` Redis) | **Blueprint = Infrastructure as Code**: khai báo nhiều resource trong một file (web service **và** Key Value/Redis), Render tạo hết khi sync |
| Build | `[build] builder = "DOCKERFILE"`, `dockerfilePath` | `runtime: docker`, `rootDir: 06-lab-complete`, `dockerfilePath`, `dockerContext` (phải chỉ rõ thư mục con vì Blueprint đọc từ gốc repo) |
| Health check | `[deploy] healthcheckPath = "/health"`, `healthcheckTimeout` | `healthCheckPath: /health` |
| Restart | `restartPolicyType = "ON_FAILURE"`, `restartPolicyMaxRetries` | Render tự restart khi health check fail; không có khoá riêng |
| Biến môi trường / secret | Không khai báo trong file; đặt bằng `railway variables set AGENT_API_KEY=...` (không vào git) | `envVars` trong file: giá trị thường ghi thẳng, secret dùng `generateValue: true` hoặc `sync: false`; `REDIS_URL` nối tự động bằng `fromService.property: connectionString` |
| Plan / region | Chọn ở dashboard | `plan: free`, `region: singapore` khai báo trong file |
| Port | Railway inject `PORT`; Dockerfile `CMD ... --port ${PORT:-8000}` | Render inject `PORT` (mặc định 10000); dùng chung `CMD` đó |

**Vì sao khác:** Railway thiên về CLI-first, file toml chỉ giữ phần build/deploy của service hiện tại, còn hạ tầng ghép bằng lệnh. Render Blueprint mô tả toàn bộ hạ tầng trong git nên tái tạo được bằng một thao tác (và auto-sync khi push), đổi lại phải khai báo kỹ hơn (rootDir, kiểu secret, liên kết Redis).

### Exercise 3.3: Pipeline Cloud Run (`03-cloud-deployment/production-cloud-run/`)

`cloudbuild.yaml` là pipeline Cloud Build 4 bước chạy tuần tự (`waitFor`), kích hoạt khi push:
1. **test** — image `python:3.11-slim`: `pip install -r requirements.txt pytest` rồi `pytest tests/`; fail là dừng cả pipeline.
2. **build** — `docker build` gắn 2 tag `gcr.io/$PROJECT_ID/ai-agent:$COMMIT_SHA` và `:latest`, dùng `--cache-from :latest` để tận dụng layer cache.
3. **push** — đẩy mọi tag lên Container Registry.
4. **deploy** — `gcloud run deploy ai-agent` với image theo `$COMMIT_SHA` (truy vết/rollback được theo commit), region `asia-southeast1`, min 1 / max 10 instance, 512Mi / 1 CPU, timeout 60 s, `ENVIRONMENT=production`, `OPENAI_API_KEY` lấy từ **Secret Manager** (`--set-secrets`).

`service.yaml` là cấu hình Knative tương đương ở dạng khai báo: `minScale 1` (tránh cold start), `maxScale 10`, `containerConcurrency 80`, giới hạn CPU/RAM, secret qua `secretKeyRef`, `livenessProbe` + `startupProbe` gọi `/health`.

---

## Part 4: API Security

### Exercise 4.1: API key (`04-api-gateway/develop`, `AGENT_API_KEY=secret-key-123 python app.py`)

- **Check ở đâu:** dependency `verify_api_key` (dòng 39) dùng `APIKeyHeader(name="X-API-Key")`, gắn vào `/ask` bằng `Depends`. Key mong đợi đọc từ env `AGENT_API_KEY` (dòng 35).
- **Sai key:** thiếu header → **401** "Missing API key"; có header nhưng sai → **403** "Invalid API key" (bản Part 6 của em trả 401 cho cả hai theo yêu cầu script chấm).
- **Rotate key:** đổi biến môi trường `AGENT_API_KEY` trên platform rồi restart/redeploy, không sửa code. Muốn không downtime thì cho phép tạm thời 2 key (cũ + mới), cập nhật client, rồi gỡ key cũ.

```text
$ curl -i -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

$ curl -i -X POST localhost:8000/ask -H "X-API-Key: wrong-key" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 403 Forbidden
{"detail":"Invalid API key."}

$ curl -i -X POST localhost:8000/ask -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 200 OK
{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 4.2: JWT (`04-api-gateway/production`)

JWT flow (`auth.py`): `POST /auth/token` kiểm tra username/password → `create_token` ký payload `{sub, role, iat, exp}` bằng HS256 với `JWT_SECRET` (hết hạn sau 60 phút) → client gửi `Authorization: Bearer <token>` → `verify_token` decode và kiểm tra chữ ký + `exp` mỗi request, **không cần tra DB**; token hết hạn → 401 "Token expired".

```text
$ curl -i -X POST localhost:8000/auth/token -H "Content-Type: application/json" -d '{"username":"student","password":"demo123"}'
HTTP/1.1 200 OK
{"access_token":"<TOKEN>","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer <TOKEN đã che>"}

$ curl -i -X POST localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Explain JWT"}'
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

$ curl -i -X POST localhost:8000/ask -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question":"Explain JWT"}'
HTTP/1.1 200 OK
{"question":"Explain JWT","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}
```

### Exercise 4.3: Rate limiting

- **Thuật toán:** **sliding window log** — mỗi user một `deque` timestamp; mỗi request bỏ các timestamp cũ hơn 60 s, nếu còn ≥ limit thì 429 (kèm `Retry-After`, `X-RateLimit-*`), ngược lại append timestamp.
- **Limit:** user thường **10 request / 60 s** (`rate_limiter_user`), admin 100 request / 60 s (`rate_limiter_admin`).
- **Bypass cho admin:** `app.py` chọn `rate_limiter_admin if role == "admin"` theo `role` trong JWT (tài khoản `teacher` có role admin) → admin có quota cao hơn; muốn bỏ hẳn thì skip `limiter.check` khi role là admin.
- Lưu ý: state nằm trong RAM của process → mất khi restart và không chia sẻ giữa nhiều instance; Part 6 em làm bằng Redis sorted set.

```text
# Request "Explain JWT" ở 4.2 đã dùng 1/10 lượt, sau đó gọi 20 lần liên tiếp:
Test  1 -> 200
...
Test  9 -> 200
Test 10 -> 429
...
Test 20 -> 429

$ curl -i -X POST localhost:8000/ask -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question":"Test 21"}'
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
retry-after: 59
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

### Exercise 4.4: Cost guard

`04-api-gateway/production/cost_guard.py` (bản demo, in-memory): mỗi user có `UsageRecord` theo ngày (input/output token → cost theo giá gpt-4o-mini), budget **$1/ngày/user** và **$10/ngày toàn hệ thống**; `check_budget()` gọi **trước** LLM: vượt budget user → **402 Payment Required**, vượt global → 503; cảnh báo log khi đạt 80 %; `record_usage()` gọi **sau** LLM để cộng chi phí thật. Output `/me/usage` sau 10 request: `{"requests":10,"input_tokens":40,"output_tokens":304,"cost_usd":0.000188,"budget_usd":1.0,"budget_remaining_usd":0.999812}`.

Cách em làm trong Part 6 (`06-lab-complete/app/cost_guard.py`), theo đúng logic của bài (budget theo tháng, lưu Redis, reset đầu tháng):

```python
def _keys():
    now = datetime.now(timezone.utc)
    return f"cost:daily:{now:%Y-%m-%d}", f"cost:monthly:{now:%Y-%m}"   # key mới mỗi ngày/tháng = tự reset

def check_budget():                      # gọi TRƯỚC khi gọi LLM
    daily, monthly = r.mget(*_keys())
    if float(daily or 0)   >= settings.daily_budget_usd:   raise HTTPException(402, "Daily budget exceeded")
    if float(monthly or 0) >= settings.monthly_budget_usd: raise HTTPException(402, "Monthly budget exceeded")

def record_cost(input_tokens, output_tokens):   # gọi SAU khi gọi LLM
    cost = input_tokens/1000*0.00015 + output_tokens/1000*0.0006
    pipe = r.pipeline()
    pipe.incrbyfloat(daily_key, cost);   pipe.expire(daily_key, 2*24*3600)
    pipe.incrbyfloat(monthly_key, cost); pipe.expire(monthly_key, 32*24*3600)
    pipe.execute()
```

- State trong **Redis** (`INCRBYFLOAT` là atomic) → mọi worker/instance dùng chung, còn sau restart.
- Key chứa ngày/tháng (`%Y-%m`) nên sang tháng mới tự dùng key mới = **reset đầu tháng**, key cũ tự hết hạn bằng `EXPIRE 32 ngày`.
- Budget đọc từ env `DAILY_BUDGET_USD` / `MONTHLY_BUDGET_USD` (mặc định $1/ngày, $10/tháng). Redis lỗi → 503 (fail closed) thay vì cho gọi LLM không kiểm soát.
- Script chấm xác nhận: budget 0.000001 USD → **402 ở request thứ 1**.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks (`05-scaling-reliability/develop`)

`/health` (liveness) trả 200 kèm uptime, version và kiểm tra bộ nhớ (`psutil`); `/ready` (readiness) trả 503 khi `_is_ready=False` (đang khởi động hoặc đang tắt), 200 khi sẵn sàng, kèm số request đang xử lý.

```text
$ curl localhost:8000/health
{"status":"ok","uptime_seconds":0.6,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T08:22:22.945851+00:00","checks":{"memory":{"status":"ok","used_percent":20.4}}}
$ curl localhost:8000/ready
{"ready":true,"in_flight_requests":1}
```

**Liveness khác readiness:** liveness hỏi "process còn sống không?" — fail thì platform **restart** container; readiness hỏi "có nhận traffic được không?" — fail thì load balancer **ngừng gửi** request nhưng không restart. Vì vậy ở Part 6 `/health` luôn 200 kể cả khi Redis chết (restart cũng không chữa được Redis), còn `/ready` trả 503 để rút instance khỏi LB cho tới khi Redis về.

### Exercise 5.2: Graceful shutdown

Uvicorn bắt SIGTERM, ngừng nhận kết nối mới, chờ request đang chạy (`timeout_graceful_shutdown=30`), rồi chạy phần shutdown của `lifespan` (`_is_ready=False`, chờ `_in_flight_requests` về 0). Handler `signal.signal(SIGTERM, …)` trong app chỉ log.

```text
$ python app.py & PID=$!
$ curl -X POST "localhost:8000/ask?question=Long%20task" &   ;   kill -TERM $PID   # gửi SIGTERM ngay 50 ms sau request
request result: {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."} [HTTP 200, 3.013671s]
process exit code: 0
--- server log ---
INFO:     Shutting down
INFO:     127.0.0.1:45076 - "POST /ask?question=Long%20task HTTP/1.1" 200 OK
INFO:     Waiting for application shutdown.
2026-11-26 15:22:26,077 INFO 🔄 Graceful shutdown initiated...
2026-11-26 15:22:26,077 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
INFO:     Finished server process [18394]
```

Request đang chạy (3 s) **vẫn hoàn thành với 200**, server log "Shutting down" trước rồi mới trả response, process thoát với exit code 0.

### Exercise 5.3: Stateless design

`05-scaling-reliability/production/app.py` lưu toàn bộ session/conversation history trong Redis (`session:<id>`, `SETEX` TTL 1 giờ, giữ tối đa 20 message) thay vì dict `conversation_history = {}` trong RAM. **Lý do:** khi scale ra nhiều instance sau load balancer, mỗi instance có RAM riêng — request 1 vào instance A lưu history ở A, request 2 vào instance B sẽ không thấy gì; restart/deploy cũng xoá sạch RAM. Đưa state ra Redis thì instance nào cũng phục vụ được bất kỳ request nào và có thể thêm/bớt instance tuỳ ý. Part 6 áp dụng cùng nguyên tắc cho rate limit và cost guard (script chấm restart agent vẫn thấy 429).

### Exercise 5.4: Load balancing (cổng 8080, `POST /chat`)

```text
$ touch .env.local
$ docker compose -p k5-day14-scaling up -d --build --scale agent=3
$ docker compose -p k5-day14-scaling ps
k5-day14-scaling-agent-1   Up (health: starting)
k5-day14-scaling-agent-2   Up (health: starting)
k5-day14-scaling-agent-3   Up (health: starting)
k5-day14-scaling-nginx-1   Up
k5-day14-scaling-redis-1   Up (healthy)

# 10 request POST /chat, đếm served_by:
      3 instance-ab221d
      4 instance-c995a4
      3 instance-f6ce46

# Failover: docker stop k5-day14-scaling-agent-2, rồi gọi thêm 10 request:
      6 200 instance-ab221d
      4 200 instance-f6ce46
```

Nginx phân phối round-robin đều cho 3 instance (3/4/3). Khi dừng 1 instance, cả 10 request sau đó vẫn trả **200**, traffic chuyển hết sang 2 instance còn lại (Nginx bỏ qua upstream không kết nối được).

### Exercise 5.5: `python test_stateless.py`

```text
Session ID: f941ad1f-45f9-4088-80ef-067a2dd57b79
Request 1: [instance-ab221d]  Q: What is Docker?
Request 2: [instance-f6ce46]  Q: Why do we need containers?
Request 3: [instance-c995a4]  Q: What is Kubernetes?
Request 4: [instance-ab221d]  Q: How does load balancing work?
Request 5: [instance-f6ce46]  Q: What is Redis used for?
------------------------------------------------------------
Total requests: 5
Instances used: {'instance-c995a4', 'instance-f6ce46', 'instance-ab221d'}
✅ All requests served despite different instances!
--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```

**Giải thích:** 5 câu của cùng một session lần lượt được 3 instance khác nhau trả lời (round-robin), nhưng khi đọc lại history vẫn đủ **10 message** (5 user + 5 assistant) theo đúng thứ tự. Mỗi instance đọc/ghi `session:<id>` trên Redis dùng chung chứ không giữ gì trong RAM, nên session không bị "dính" vào một instance — đây chính là stateless. Nếu lưu trong RAM thì mỗi instance chỉ thấy 1–2 câu của mình. Xong thì `docker compose -p k5-day14-scaling down -v`.

---

## Part 6: Final Project — output `check_production_ready.py`

Chạy đầy đủ (có Docker) trong `06-lab-complete/`: `python check_production_ready.py` → **`Result: 12/12 tiêu chí đạt · 43/43 checks passed`** (image 273 MB, `docker stop` exit 0 trong 0.6 s).

```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

---

## AI log

Bài này được làm cùng **Claude Code** (Claude Opus 5.5), chạy trong terminal WSL2 của em với quyền đọc/sửa repo, chạy lệnh và gọi Render. Em đưa yêu cầu, xem output từng bước, và kiểm tra lại kết quả. Mọi con số trong bài do lệnh chạy thật sinh ra (có trong `reports/`, `benchmark-results/`, `screenshots/` và output dán ở trên), không có số nào do AI tự điền.

| # | Hỏi / giao gì | AI trả về / làm gì | Em tự kiểm tra thế nào |
|---|---|---|---|
| 1 | Giải thích bài lab Day 14 | Tóm tắt README, RUBRIC, SUBMISSION, các lưu ý của K5 | Đối chiếu với README/RUBRIC |
| 2 | Kiểm tra và dọn các service Render cũ | Liệt kê 2 service cũ (đều đang suspended); công cụ Render không có lệnh xoá nên em tự xoá trên dashboard | AI liệt kê lại workspace và thấy trống |
| 3 | Viết code Part 6 (điền TODO) | Đọc `check_production_ready.py` để nắm tiêu chí rồi viết `config.py`, `auth.py`, `redis_client.py`, `rate_limiter.py` (Redis sorted set), `cost_guard.py` (Redis theo ngày/tháng), `main.py` (log JSON, `/health`, `/ready`, lifespan), Dockerfile multi-stage non-root, compose, `.dockerignore`, `.env.example`, `render.yaml`, `railway.toml` | Chạy `python check_production_ready.py` bản đầy đủ có Docker → **PASS 12/12** ngay lần đầu; đọc lại diff |
| 4 | Commit, push, deploy lên Render | Push lên `main`, tạo Render Blueprint từ `06-lab-complete/render.yaml` (web + Redis free), chờ live, chạy các lệnh tự kiểm, viết `DEPLOYMENT.md` | Em tự lấy `AGENT_API_KEY` trên dashboard (key không đi qua AI) và chạy lệnh `/ask` có key → **200** |
| 5 | Part 7 | Chạy load test, eval, stack Docker, Locust, các thí nghiệm A–E; ghi thời gian alert bằng script poll `/api/v1/alerts`; chụp ảnh bằng headless Chrome (Playwright); thêm 2 entry KB (JWT, HPA) để v2 đạt 23/23; sửa lỗi `PYTHONUSERBASE` trong `production/Dockerfile` của đề (agent không tìm thấy uvicorn); viết job CI và `BENCHMARK_REPORT.md` | Đối chiếu số trong report với file JSON/CSV; xem lại 4 ảnh; eval `--baseline` báo 0 regression |
| 6 | Part 1–5 | Chạy từng bản develop/production, build image, Compose, test 401/403/200/429, SIGTERM, scale 3 + failover; tổng hợp output thật vào file này | Đối chiếu output với lệnh trong `CODE_LAB.md`; số MB lấy từ `docker images` / `docker image inspect` |

Những chỗ AI phát hiện và tự sửa trong lúc làm: (1) venv chung không có pip → tạo `.venv` riêng bằng `uv`; (2) Jaeger v2 không có `/api/traces` → chuyển sang `/api/v3/traces`; (3) Grafana chụp ra panel trống vì panel chưa render → chụp lại với viewport cao hơn và chờ lâu hơn; (4) ảnh trace lỗi chưa mở rộng được phần event `exception` → giữ ảnh có span đỏ, HTTP 503, `error.type` và "Events (1)", nội dung exception được dán ở dạng log JSON trong `BENCHMARK_REPORT.md`.
