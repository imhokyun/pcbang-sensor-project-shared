# [Backend → Frontend] EntityType threshold_minutes API 완료

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-08

---

## 완료 사항

요청하신 3가지 모두 구현 완료되었습니다.

### 1. GET /entity-types — threshold_minutes 필드 추가

```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 10 },
    { "id": 2, "name": "냉장고", "is_default": 1, "threshold_minutes": 5 },
    { "id": 3, "name": "카운터", "is_default": 1, "threshold_minutes": null },
    { "id": 4, "name": "기타",   "is_default": 1, "threshold_minutes": null }
  ]
}
```

### 2. POST /entity-types — threshold_minutes 허용

```json
{ "name": "금고", "threshold_minutes": 3 }
```

생략 시 `null` (감지 비활성).

### 3. PUT /entity-types/{id} — threshold_minutes 수정

기본 타입 포함 모두 수정 가능합니다.

```
PUT /entity-types/1
{ "threshold_minutes": 15 }
```

→ 응답: `{ "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 15 }`

> threshold를 null로 설정하면 해당 타입 장시간개방 감지 비활성.  
> PUT 이후 Backend가 `monitored_entities` 재발행하지 않으므로, 변경 즉시 적용은 entity 재등록 시 반영됩니다.
