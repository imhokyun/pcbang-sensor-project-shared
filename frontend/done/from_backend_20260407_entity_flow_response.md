# [Backend → Frontend] Entity 상태 → Alert 흐름 및 CORS 수정 완료

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

---

## 처리 결과 요약

### 1. CORS on 500 errors — 수정 완료

FastAPI에 generic exception handler를 추가했습니다. 이제 500 에러 응답에도 CORS 헤더가 포함됩니다.

### 2. `entity.update` 브로드캐스트 — 이미 구현됨

MQTT에서 entity 상태 변화 수신 시, `triggers_alert` 여부와 무관하게 `entity.update`를 WS로 브로드캐스트하고 있습니다:

```json
{
  "type": "entity.update",
  "store_id": 30584,
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "custom_name": "정문 센서",
  "type_name": "motion_sensor",
  "state": "on",
  "timestamp": "2026-04-07T03:43:19.390388+00:00"
}
```

### 3. `alert.new` 페이로드 — `entity_name`, `message` 필드 추가

Frontend 스펙에 맞게 필드를 추가했습니다. 현재 `alert.new` 페이로드:

```json
{
  "type": "alert.new",
  "alert_id": 1,
  "store_id": 30584,
  "store_name": "테스트매장",
  "importance": 3,
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "entity_name": "정문 센서",
  "custom_name": "정문 센서",
  "type_name": "motion_sensor",
  "state_from": "off",
  "state_to": "on",
  "message": "정문 센서 on",
  "stream_url": null,
  "is_in_schedule": true,
  "timestamp": "2026-04-07T03:43:19.390388+00:00"
}
```

`init.pending_alerts`도 동일한 형식으로 정렬했습니다.

---

## Alert 생성 흐름 (구현 완료)

```
MQTT 수신 (pcbang/{store_id}/entities/{ha_entity_id}/state)
  └→ entity.update WS 브로드캐스트 (모든 entity)
  └→ DB에서 entity 조회
       └→ triggers_alert = 1 AND alert_triggers 매칭
            └→ alert_events INSERT
                 └→ alert.new WS 브로드캐스트
```

---

## 확인 사항

Edge에서 MQTT를 발행할 때 해당 entity의 `triggers_alert=1`이어야 alert가 생성됩니다.

현재 seed 데이터 기준:
- store 30584의 entity가 등록되어 있고 `triggers_alert=1`이면 자동 alert 생성됩니다.

Entity 등록 시 `type_id=0` 오류는 별도 메시지로 안내했습니다.  
→ `from_backend_20260407_entity_type_id_bug.md` 참조
