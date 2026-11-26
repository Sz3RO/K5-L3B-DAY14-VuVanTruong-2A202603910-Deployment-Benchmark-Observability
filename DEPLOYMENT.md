# Deployment Information

## Public URL
https://day14-production-agent-b6ln.onrender.com

## Platform
**Render** — deploy bằng Blueprint từ [`06-lab-complete/render.yaml`](06-lab-complete/render.yaml)
(Blueprint Path = `06-lab-complete/render.yaml`, branch `main`, auto-sync khi push).

| Resource | Loại | Plan | Region |
|---|---|---|---|
| `day14-production-agent-b6ln` | Web Service (Docker, `rootDir: 06-lab-complete`) | Free | Singapore |
| `day14-agent-redis-b6ln` | Key Value (Redis) | Free | Singapore |

Health check của Render: `GET /health`. Agent dùng Redis cho rate limit và cost guard; `/ready` trả 200 khi kết nối được Redis.

> Free plan ngủ sau ~15 phút không có request — request đầu tiên có thể mất ~30–60 giây để đánh thức.

## Test Commands

Đặt biến (lấy `AGENT_API_KEY` ở Render Dashboard → service → Environment; không ghi key vào repo):

```bash
export BASE=https://day14-production-agent-b6ln.onrender.com
export AGENT_API_KEY=<key trên Render>
```

### Tự kiểm CP6 (chỉ in mã HTTP)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

### Health Check
```bash
curl $BASE/health
# {"status":"ok","environment":"production"}
```

### API Test (with authentication)
```bash
curl -X POST $BASE/ask \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
# {"question":"Hello","answer":"..."}
```

### Script chấm
```bash
cd 06-lab-complete
python check_production_ready.py --public-url https://day14-production-agent-b6ln.onrender.com
# ✅ Public URL: GET .../health → 200
```

## Environment Variables Set
Đặt trên Render (qua Blueprint), **không** ghi trong code:

| Biến | Giá trị / nguồn |
|---|---|
| `PORT` | Render tự cấp; `CMD` của Dockerfile đọc `${PORT:-8000}` |
| `ENVIRONMENT` | `production` |
| `AGENT_API_KEY` | `generateValue: true` — Render sinh ngẫu nhiên (secret) |
| `REDIS_URL` | `fromService` → connection string của Key Value `day14-agent-redis-b6ln` |
| `RATE_LIMIT_PER_MINUTE` | `10` |
| `DAILY_BUDGET_USD` | `1.0` |
| `MONTHLY_BUDGET_USD` | `10.0` |
| `LOG_LEVEL` | không đặt → mặc định `INFO` |

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png) — Render Dashboard: service `Live`, Blueprint managed, 3 deploy thành công
- [Service running](screenshots/running.png) — trạng thái lấy qua Render API (deploy LIVE, Redis available, log JSON của app) + `/health`, `/ready` gọi từ URL public
- [Test results](screenshots/test.png) — 4 lệnh tự kiểm CP6 trên URL public: 200 / 200 / 401 / 200 (key đã che)
