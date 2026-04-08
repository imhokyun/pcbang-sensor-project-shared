# [Edge → Backend] A4-Edge 구현 완료 — 연동 준비됨

**발신**: Edge  
**수신**: Backend  
**일시**: 2026-04-08

---

## 결론

**A4-Edge 구현이 완료되어 Backend A4-BE와 연동 준비가 됐습니다.**

---

## 구현 완료 항목

- `coordinator.py`에 asyncio 타이머 추가
- `monitored_entities` object[] 형식 파싱 (`threshold_minutes` 추출)
- string[] 구형 형식 하위 호환 유지 (Backend 업데이트 전 임시 동작 가능)
- 테스트 47/47 통과

---

## Edge에서 발행할 MQTT 토픽

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

## Backend에 필요한 작업

### 1. monitored_entities payload 형식 변경 (필수)

현재 Backend가 발행하는 형식:
```json
{ "entities": ["binary_sensor.naengjanggo_1", ...] }
```

변경 후 (threshold_minutes 포함):
```json
{
  "entities": [
    { "ha_entity_id": "binary_sensor.naengjanggo_1", "threshold_minutes": 5 },
    { "ha_entity_id": "binary_sensor.jeongmun_senseo", "threshold_minutes": 10 }
  ]
}
```

> Edge는 threshold가 없는 entity에 대해 타이머를 시작하지 않습니다.
> 현재 string[] 형식을 수신해도 동작하지만 장시간 개방 감지는 비활성화됩니다.

### 2. long_open 토픽 구독 추가

`pcbang/{store_id}/entities/{ha_entity_id}/long_open` 구독 후 AlertEvent 생성.

---

## 통합 테스트 협조

Backend A4-BE 완료 통보 시:
1. HA에서 monitored binary_sensor를 수동으로 `on` 설정
2. `threshold_minutes` 경과 후 `long_open` 토픽 발행 확인
3. Backend AlertEvent 생성 → Frontend 알림 수신 확인

결과를 `shared/to_orchestrator/`에 공동 보고하겠습니다.
