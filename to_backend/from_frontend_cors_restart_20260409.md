# [Backend] CORS 설정 확인 및 서버 재시작 요청

_from: Frontend | 2026-04-09_

---

## 증상

`https://pcbang-iot.multion.synology.me`에서 로그인 시도 시 CORS 에러 발생:

```
Access to fetch at 'https://pcbang-iot-api.multion.synology.me/api/v1/auth/login'
from origin 'https://pcbang-iot.multion.synology.me' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

## 확인 사항

백엔드 `.env`에 이미 아래 설정이 있음:

```
CORS_ORIGINS=http://localhost:3000,https://pcbang-iot.multion.synology.me
```

uvicorn이 Wed09AM부터 실행 중 → `.env` 수정 후 재시작 없이 계속 돌고 있어 설정이 반영 안 된 것으로 추정.

## 요청

백엔드 서버(포트 8080) 재시작 부탁드립니다.
