# Entity 생성 시 type_id=0 오류

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

---

## 문제

Frontend에서 Entity를 생성할 때 `type_id=0`을 전송하고 있습니다.

Backend 로그에서 다음 오류가 반복적으로 발생했습니다:

```
sqlalchemy.exc.IntegrityError: (asyncpg.exceptions.ForeignKeyViolationError)
Key (type_id)=(0) is not present in table "entity_types"
```

## 원인

`entity_types` 테이블의 ID는 1부터 시작합니다. `0`은 유효하지 않은 값입니다.

## 조치 요청

Entity 생성 폼에서 type_id 기본값 또는 선택 값이 `0`이 되지 않도록 수정 필요합니다.

- Entity 생성 전 반드시 `GET /api/v1/entity-types`를 호출하여 유효한 타입 목록을 가져와야 합니다.
- 폼의 초기값을 빈 값(`null`/`undefined`) 또는 목록의 첫 번째 항목으로 설정하세요.
- `type_id=0` 또는 미선택 상태로 제출되지 않도록 폼 유효성 검사를 추가해 주세요.

## 참고

현재 seed된 entity_types:

| id | name |
|----|------|
| 1  | motion_sensor |
| 2  | door_sensor |
| 3  | relay |
| 4  | camera |

API: `GET /api/v1/entity-types` → `{ success: true, data: [...] }`
