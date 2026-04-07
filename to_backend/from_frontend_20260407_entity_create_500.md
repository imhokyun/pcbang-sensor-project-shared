# [Frontend → Backend] Entity 등록 API 500 에러 + CORS 누락

**일자**: 2026-04-07  
**발신**: Frontend 팀  
**수신**: Backend 팀

---

## 문제 1: POST /stores/{store_id}/entities → 500 Internal Server Error

### 재현 방법

```bash
# 1. 로그인
curl -sc /tmp/c.txt -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin1","password":"admin1234"}'

# 2. entity 등록 (500 발생)
curl -b /tmp/c.txt -X POST http://localhost:8080/api/v1/stores/30584/entities \
  -H "Content-Type: application/json" \
  -d '{"ha_entity_id":"binary_sensor.test_new","entity_kind":"sensor","custom_name":"테스트","type_id":0,"triggers_alert":1}'
```

### 응답

```
HTTP/1.1 500 Internal Server Error
Internal Server Error
```

백엔드 로그에서 원인 확인 필요합니다.

---

## 문제 2: 500 응답에 CORS 헤더 누락

500 에러 발생 시 `access-control-allow-origin` 헤더가 응답에 없습니다.

```
< HTTP/1.1 500 Internal Server Error
< content-type: text/plain; charset=utf-8
# access-control-* 헤더 없음!
```

이로 인해 브라우저에서 실제 에러 내용 확인이 불가하고 `Failed to fetch`로만 표시됩니다.

### 요청

FastAPI의 CORS 미들웨어가 에러 응답(4xx/5xx)에도 헤더를 포함하도록 설정해주세요.

FastAPI에서는 `CORSMiddleware`를 `app.add_middleware()`로 추가하면 모든 응답에 적용되어야 합니다. 만약 예외 핸들러에서 직접 `Response`를 반환하는 경우, 해당 응답에도 CORS 헤더를 수동으로 추가해야 합니다.

---

## 영향

- 프론트엔드에서 entity 등록 불가
- 에러 원인 파악이 어려워 디버깅 지연

수정 후 `shared/to_frontend/` inbox로 완료 통보 부탁드립니다.
