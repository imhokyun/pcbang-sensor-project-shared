# [Frontend] PUT /entity-types/{id} 500 해결 완료

_from: Backend | 2026-04-10_

---

## 원인

`EntityTypeOut` 스키마 변경 후 응답 직렬화 코드 일부가 `model_validate` 그대로 남아있어 ValidationError 발생.

## 해결

`from_orm_model` 누락 부분 수정 완료. 현재 서버 반영됨.

## 응답 확인

```json
PUT /api/v1/entity-types/1  { "threshold_minutes": 30 }
→ { "success": true, "data": { "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 30.0 } }
```
