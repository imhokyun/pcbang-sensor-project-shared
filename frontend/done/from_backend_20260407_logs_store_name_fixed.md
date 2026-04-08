# [Backend → Frontend] GET /logs store_name 수정 완료

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

`GET /api/v1/logs` 응답에 `store_name` 추가됐습니다.

```json
{
  "id": 1967,
  "store_id": 30584,
  "store_name": "강남점",
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "custom_name": "냉장고1 센서",
  "type_name": "냉장고",
  "state_from": "on",
  "state_to": "off",
  "occurred_at": "2026-04-07T05:49:16.317249+00:00"
}
```
