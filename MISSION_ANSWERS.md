#  Delivery Checklist — Day 14 Lab Submission

> **Student Name:** Đinh Trần Nguyên
> **Student ID:** 2A202603517
> **Date:** 25/11/2026

---

# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. Hardcode API key và thông tin DB trong code (OPENAI_API_KEY, DATABASE_URL ở dòng 17-18).
2. Gán cứng cấu hình DEBUG = True và MAX_TOKENS = 500 trong code thay vì dùng biến môi trường (dòng 21-22).
3. Dùng print() thay vì structured logging và in cả OPENAI_API_KEY ra log (dòng 33-34, 38).
4. Không có endpoint kiểm tra sức khỏe như /health hay /ready (dòng 42-43).
5. Cấu hình server cố định host=localhost, port=8000, reload=True (dòng 49-54) và không xử lý graceful shutdown.

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trong code (dòng 17-22, 51-53) | Đọc từ biến môi trường qua settings trong config.py (dòng 24, 196-198) | Không lộ secret khi push Git và đổi cấu hình theo môi trường không cần sửa code. |
| Health check | Không có (dòng 42-43) | Có /health, /ready và /metrics (dòng 135-173) | Nền tảng biết khi nào cần restart container (/health) và khi nào được route traffic (/ready). |
| Logging | print() thường, log cả API key (dòng 33-38) | Structured JSON logging, không log secret (dòng 28-32, 111-122) | Dễ thu thập, tìm kiếm trên hệ thống log tập trung và tránh lộ thông tin nhạy cảm. |
| Shutdown | Tắt đột ngột | Có lifespan và xử lý SIGTERM (dòng 40-67, 180-189) | Cho request đang chạy hoàn thành trước khi tắt container, tránh lỗi cho người dùng. |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: python:3.11
2. Working directory: /app
3. Copy requirements.txt trước COPY . .: Để tận dụng Docker layer cache; khi sửa source code nhưng không đổi dependencies thì không phải chạy lại pip install
4. CMD vs ENTRYPOINT: CMD đặt lệnh mặc định và dễ bị ghi đè khi chạy docker run; ENTRYPOINT cố định lệnh thực thi chính, tham số truyền thêm sẽ nối vào sau.

### Exercise 2.2: Container test
- Kết quả gọi POST /ask?question=What+is+Docker: {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}

### Exercise 2.3: Image size comparison
- Develop: 1670 MB
- Production: 240 MB
- Difference: 85%

### Exercise 2.4: Docker Compose stack
- Sơ đồ các service (mạng internal): Client -> Nginx (8088:80) -> Agent (8000) -> Redis (6379) & Qdrant (6333)
- Kết quả GET /health: {"status":"ok","uptime_seconds":166.5,"version":"2.0.0","timestamp":"2026-11-25T09:08:55.609435"}
- Kết quả POST /ask: {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://d14larb-production.up.railway.app
- Screenshot: screenshots/dashboard.png, screenshots/running.png, screenshots/test.png
- Kết quả 4 lệnh tự kiểm tra: /health -> 200, /ready -> 200, /ask (thiếu key) -> 401, /ask (có key) -> 200.

### So sánh railway.toml và render.yaml
- railway.toml dùng định dạng TOML, cấu hình ngắn gọn cho 1 service ([build] builder = "DOCKERFILE", [deploy] healthcheckPath = "/health"); biến môi trường và Redis được gắn qua Dashboard hoặc CLI.
- render.yaml là Blueprint YAML (IaC cho toàn stack), khai báo danh sách nhiều services (web, database/Redis), loại service (type: web, runtime: docker), healthCheckPath và cả envVars (secret dùng generateValue: true hoặc sync: false để không lộ trong Git).

### Exercise 3.3: Cloud Run pipeline (cloudbuild.yaml và service.yaml)
- cloudbuild.yaml chạy pipeline 4 bước khi push lên main: (1) test bằng pytest trên python:3.11-slim, (2) build Docker image gắn tag $COMMIT_SHA và latest, (3) push lên Google Container Registry, (4) deploy lên Cloud Run (asia-southeast1, min-instances=1, max-instances=10, lấy secret từ Secret Manager).
- service.yaml định nghĩa cấu hình Knative/Cloud Run tương ứng: giới hạn concurrency 80 request/instance, tài nguyên 1 CPU / 512Mi RAM, livenessProbe ở /health và startupProbe ở /ready.

## Part 4: API Security

### Exercise 4.1-4.3: Test results
- Exercise 4.1 (không có API key -> 401):
  HTTP/1.1 401 Unauthorized
  {"detail":"Missing API key. Include header: X-API-Key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3OTU1OTgxNjIsImV4cCI6MTc5NTYwMTc2Mn0.fynncooEP3VG9XQyh2B_Ql29DiN9obN1eO_vMoK74r8D"}
- Exercise 4.1 (có X-API-Key -> 200):
  HTTP/1.1 200 OK
- Exercise 4.2 (JWT Bearer token, đã che token -> 200):
  HTTP/1.1 200 OK
  {"question":"Explain JWT","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}
- Exercise 4.3 (vượt rate limit -> 429):
  Request 1-10: HTTP/1.1 200 OK
  Request 11-12:
  HTTP/1.1 429 Too Many Requests
  x-ratelimit-limit: 10
  x-ratelimit-remaining: 0

### Exercise 4.4: Cost guard implementation
Theo dõi số token input/output sau mỗi lần gọi LLM (record_usage) để cộng dồn chi phí USD theo ngày/tháng cho từng user và toàn hệ thống. Trước khi xử lý request (check_budget), kiểm tra nếu vượt ngân sách user thì trả về 402 Payment Required, vượt ngân sách hệ thống trả 503, và ghi log cảnh báo khi đạt 80%. Trên production, bộ đếm chi phí được lưu trong Redis để dùng chung giữa nhiều instance.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
- Health check: /health (liveness) kiểm tra tiến trình còn sống; /ready (readiness) kiểm tra kết nối Redis/DB, trả 503 nếu chưa sẵn sàng nhận traffic.
- Graceful shutdown: bắt tín hiệu SIGTERM để hoàn thành request đang xử lý trước khi tắt container.
- Stateless & Load Balancing (--scale agent=3): chạy test_stateless.py gửi 5 request được Nginx phân tải qua 3 instance (instance-35e700, instance-571fc7, instance-162090), lịch sử hội thoại 10 messages vẫn giữ nguyên nhờ lưu state trong Redis thay vì RAM.

## Part 6: Final Project - Checklist Deliverable

Kết quả chạy `python check_production_ready.py` trong `06-lab-complete/` (chế độ đầy đủ, có Docker):
- Result: 12/12 tiêu chí đạt · 41/41 checks passed (PASS 12/12)
- Đạt đủ 12 hạng mục: Dockerfile (multi-stage, < 500 MB), docker-compose.yml (agent + redis), .dockerignore, Health check (/health), Readiness (/ready), API Key authentication, Rate limiting (Redis), Cost guard (Redis), Config từ environment variables, Structured JSON logging, Graceful shutdown, Public URL ready (railway.toml + PORT).

## AI Usage Log
- Công cụ: Trợ lý AI trong IDE (Antigravity).
- Mục đích: Hướng dẫn tuần tự từng checkpoint, giải thích lỗi môi trường (WSL2, Docker port/credentials, PYTHONUSERBASE), hỗ trợ hoàn thiện các TODO trong `06-lab-complete/` và tổng hợp số liệu từ JSON/CSV vào báo cáo. Toàn bộ lệnh test, deploy Railway và chạy benchmark đều được thực thi và kiểm chứng trực tiếp trên máy.
