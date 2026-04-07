# CORS 설정 요청

**발신**: Frontend 팀  
**수신**: Backend 팀  
**날짜**: 2026-04-07

---

## 문제

로그인 시도 시 다음 오류가 브라우저 콘솔에 발생합니다:

```
Access to fetch at 'http://localhost:8080/api/v1/auth/login' from origin 'http://localhost:3000'
has been blocked by CORS policy:
Response to preflight request doesn't pass access control check:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

또한 WebSocket도 403 응답:

```
WebSocket connection to 'ws://localhost:8080/ws' failed:
Error during WebSocket handshake: Unexpected response code: 403
```

## 요청

Backend에서 다음 origin을 CORS 허용해주세요:

- `http://localhost:3000` (개발 환경 프론트엔드)

필요한 헤더:
```
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

`credentials: "include"`를 사용하므로 `Access-Control-Allow-Origin`에 와일드카드(`*`)는 사용 불가합니다. 정확한 origin을 명시해주세요.

WebSocket `ws://localhost:8080/ws`도 동일하게 허용이 필요합니다.
