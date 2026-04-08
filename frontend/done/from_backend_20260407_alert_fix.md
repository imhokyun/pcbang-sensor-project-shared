# [Backend → Frontend] Alert 미생성 원인 파악 및 수정 완료

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

---

## 원인

`alert_triggers` 테이블의 상태값이 `"open"/"closed"`로 되어 있었으나,  
HA `binary_sensor`가 발행하는 실제 상태값은 `"on"/"off"`입니다.

```
alert_triggers (수정 전): state_from="closed", state_to="open"
Edge MQTT payload:        "state": "on"   ← 매칭 실패 → alert 미생성
```

## 수정 사항

`alert_triggers` 상태값을 `"off"→"on"`으로 수정했습니다.

```
alert_triggers (수정 후):
  출입문 (type_id=1): state_from="off", state_to="on"
  냉장고 (type_id=2): state_from="off", state_to="on"
```

서버가 재시작되었습니다. 즉시 테스트 가능합니다.

---

## 현재 Alert 생성 조건

두 조건을 **모두** 충족해야 `alert_events` INSERT 및 `alert.new` WS 이벤트 발생:

1. `store_entities.triggers_alert = 1`
2. `alert_triggers` 테이블에 해당 `type_id`의 `off→on` 전환이 등록되어 있어야 함

현재 등록된 entity 중 alert 대상:
- `binary_sensor.jeongmun_senseo` (출입문, triggers_alert=1) ✅
- `binary_sensor.naengjanggo_1` (냉장고, triggers_alert=1) ✅
- `binary_sensor.naengjanggo2_senseo` (냉장고, triggers_alert=1) ✅

---

## 확인 방법

Edge에서 `binary_sensor.jeongmun_senseo` 상태를 `"off"→"on"`으로 변경하면:
1. WS `entity.update` 수신 (기존과 동일)
2. WS `alert.new` 수신 (이제 정상 동작)
3. `GET /api/v1/alerts` 응답에 레코드 추가 확인
