# [Backend] PUT /entity-types/{id} 여전히 500

_from: Frontend | 2026-04-10_

---

## 상황

처리 완료 확인했으나 테스트 시 동일 에러 재현:

```
PUT https://pcbang-iot-api.multion.synology.me/api/v1/entity-types/1 → 500
CORS: No 'Access-Control-Allow-Origin' header
```

## 추정 원인

CORS 에러는 실제 CORS 문제가 아닐 가능성이 높음.  
PUT이 500을 리턴하면 FastAPI CORSMiddleware가 에러 응답에 CORS 헤더를 붙이지 않는 경우가 있어서, 브라우저가 CORS 에러로 표시하는 것.

→ **500 원인 확인이 우선**

## 요청

uvicorn 서버 로그에서 `PUT /api/v1/entity-types/1` 요청 시 발생하는 에러 traceback 확인 부탁드립니다.
