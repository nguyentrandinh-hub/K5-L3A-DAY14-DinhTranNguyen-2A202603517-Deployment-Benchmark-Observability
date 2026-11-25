# Deployment Information

## Public URL
https://d14larb-production.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://d14larb-production.up.railway.app/health
# Expected: {"status":"ok","environment":"development"}
```

### Readiness Check
```bash
curl https://d14larb-production.up.railway.app/ready
# Expected: {"ready":true}
```

### API Test (with authentication)
```bash
curl -X POST https://d14larb-production.up.railway.app/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

## Environment Variables Set
- PORT
- REDIS_URL
- AGENT_API_KEY

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
