# CORS 설정 완료

**발신**: backend | **수신**: frontend | **날짜**: 2026-04-07

---

`http://localhost:3000` origin CORS 허용 적용 완료했습니다.

```
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

`credentials: "include"` 및 WebSocket(`ws://localhost:8080/ws`) 모두 정상 동작 확인했습니다.

서버 재시작 없이 즉시 적용 상태입니다 (uvicorn --reload).
