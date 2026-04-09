# [Edge] 장시간 개방 임계값 — per-entity 수신 구조 확인 요청

_from: Backend | 2026-04-10_

---

## 현황

MQTT `monitored_entities` payload는 현재 계약상 per-entity 구조입니다:

```json
{
  "store_id": 30588,
  "entities": [
    {"ha_entity_id": "binary_sensor.door_01", "threshold_minutes": 10},
    {"ha_entity_id": "binary_sensor.fridge_01", "threshold_minutes": 5},
    {"ha_entity_id": "switch.relay_01", "threshold_minutes": null}
  ],
  "timestamp": "..."
}
```

## 확인 요청

1. **현재 구현 상태**: Edge(Custom Component)에서 `threshold_minutes`를 실제로 읽어서
   asyncio 타이머에 적용하고 있는지? 아직 미구현이라면 언제쯤 가능한지?

2. **null 처리**: `threshold_minutes: null`인 경우 장시간 개방 감지 비활성으로 처리하고 있는지?

3. **재수신 시 동작**: `monitored_entities`가 업데이트되어 재수신될 때(threshold 변경 등),
   진행 중인 타이머를 새 값으로 재설정하는지? 아니면 다음 상태 변경부터 적용되는지?

## 배경

Backend에서 장시간 개방 임계값을 매장별로 관리하도록 DB 스키마를 재설계할 예정입니다.
변경 후에도 MQTT payload 구조(`entities: [{ha_entity_id, threshold_minutes}]`)는 동일하게 유지됩니다.
Edge 측 변경사항이 있다면 알려주세요.
