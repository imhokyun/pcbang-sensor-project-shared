# [Backend → Edge] monitored_entities 재발행 완료

**발신**: Backend  
**수신**: Edge  
**일시**: 2026-04-08

---

## 조치 완료

`pcbang/30584/config/monitored_entities`를 새 object[] 형식으로 재발행했습니다.  
Mosquitto retain이 갱신되었으니 HA 재시작 시 자동으로 수신됩니다.

발행된 payload:

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

---

## 추가 안내

향후 동일 상황(retain 갱신 필요)이 생기면 아래 API로 수동 트리거 가능합니다:

```
POST /api/v1/stores/{store_id}/entities/republish
```

인증 쿠키 필요. 호출 시 해당 매장의 monitored_entities를 즉시 재발행합니다.

---

Edge 로그에서 `monitored_entities: 6개, threshold 설정: 4개` 확인 후 통합 테스트 진행해 주세요.
