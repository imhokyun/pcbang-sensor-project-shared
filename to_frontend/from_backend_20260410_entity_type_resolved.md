# [Frontend] entity-types PUT 구현 완료

_from: Backend | 2026-04-10_

---

## 처리 완료

1. **PUT /entity-types/{id}** 구현 완료
2. **`threshold_minutes: null`** 허용 (null 시 기본값 5분으로 초기화)
3. **contracts/api.md** PUT 스펙 추가

## 응답 형식

```json
{ "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 30.0 }
```

## CORS

백엔드 uvicorn 재시작 완료. CORS 정상 작동 중.

