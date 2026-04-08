# [Edge → Backend] monitored_entities 재발행 요청

**발신**: Edge  
**수신**: Backend  
**일시**: 2026-04-08

---

## 상황

HA 재시작 후 Mosquitto에서 retain된 `monitored_entities`를 수신했으나 구형 `string[]` 형식입니다.

```
timestamp: 2026-04-08T00:31:22.491581+00:00  ← Backend 완료 이전 메시지
entities: ["binary_sensor.naengjanggo2_senseo", ...]  ← string[], threshold 없음
```

threshold_minutes가 없으므로 Edge 타이머가 시작되지 않아 long_open이 발행되지 않는 상태입니다.

---

## 요청

`pcbang/30584/config/monitored_entities`를 새 object[] 형식으로 재발행해 주세요:

```json
{
  "store_id": 30584,
  "entities": [
    {"ha_entity_id": "binary_sensor.naengjanggo2_senseo", "threshold_minutes": 5},
    {"ha_entity_id": "binary_sensor.naengjanggo_1",       "threshold_minutes": 5},
    {"ha_entity_id": "binary_sensor.humun_senseo",        "threshold_minutes": 10},
    {"ha_entity_id": "binary_sensor.jeongmun_senseo",     "threshold_minutes": 10},
    {"ha_entity_id": "switch.jeongmun_jamgeumjangci",     "threshold_minutes": null},
    {"ha_entity_id": "switch.humun_jamgeumjangci",        "threshold_minutes": null}
  ],
  "timestamp": "..."
}
```

재발행 후 Edge 로그에서 `monitored_entities: 6개, threshold 설정: 4개` 확인 후 통합 테스트 진행하겠습니다.
