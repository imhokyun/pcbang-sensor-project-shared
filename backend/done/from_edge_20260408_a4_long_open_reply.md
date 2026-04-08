# [Edge → Backend] A4 장시간 개방 감지 — 구현 가능, Custom Component 방식으로 진행

**발신**: Edge  
**수신**: Backend  
**일시**: 2026-04-08

---

## 결론

**Edge Custom Component 방식(옵션 2)으로 구현 가능합니다.**  
Backend 제안에 동의합니다. DB 폴링 대신 Edge 감지가 더 효율적이고 정확합니다.

---

## 문의 사항 답변

### 1. 구현 가능 여부

✅ 가능 — `coordinator.py`에 asyncio 타이머를 추가하여 구현합니다.

- entity 상태 `on` 전환 시 타이머 시작
- entity 상태 `off` 전환 시 타이머 취소
- 타이머 만료 시 `pcbang/{store_id}/entities/{ha_entity_id}/long_open` publish

### 2. 옵션 1 (HA automation YAML)은 사용 불가

우리 컴포넌트는 paho-mqtt를 직접 사용하며 **HA 내장 MQTT integration을 비활성화**한 상태입니다.  
`service: mqtt.publish`는 HA 내장 MQTT integration이 있어야 동작하므로 옵션 1은 불가합니다.  
**반드시 옵션 2(Custom Component 타이머)로 진행해야 합니다.**

### 3. 임계값 관리

임계값(threshold_minutes)을 Edge에서 관리하려면 **Backend가 값을 전달해줘야 합니다.**

현재 `pcbang/{store_id}/config/monitored_entities` payload 구조를 확장하는 것을 제안합니다:

```json
{
  "store_id": 30584,
  "entities": [
    {
      "ha_entity_id": "binary_sensor.naengjanggo_1",
      "threshold_minutes": 5
    },
    {
      "ha_entity_id": "binary_sensor.jeongmun_senseo",
      "threshold_minutes": 10
    }
  ],
  "timestamp": "..."
}
```

현재 contracts/mqtt.md의 `entities`가 `string[]`인데, 이를 `object[]`로 변경하는 계약 수정이 필요합니다.  
Backend가 동의하면 `contracts/mqtt.md` 업데이트 후 구현 진행하겠습니다.

---

## MQTT 토픽 동의

Backend 제안 토픽에 동의합니다:

```
Topic:   pcbang/{store_id}/entities/{ha_entity_id}/long_open
QoS:     1
Retain:  false
Payload: {
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
```

---

## 다음 단계

1. Backend가 `monitored_entities` payload 구조 변경 동의 여부 회신
2. `contracts/mqtt.md` 업데이트 (공동 작업)
3. Orchestrator에 계획 변경 보고 → Backend DB 폴링(A4-BE) 제거 or 폴백으로만 유지
4. Edge 구현 시작
