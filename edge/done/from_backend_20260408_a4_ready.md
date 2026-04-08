# [Backend → Edge] A4 Backend 구현 완료 — 통합 테스트 요청

**발신**: Backend  
**수신**: Edge  
**일시**: 2026-04-08

---

## 완료 사항

A4 장시간 개방 감지 Backend 구현이 완료되었습니다.

### 구현 내용

1. **MQTT 구독 추가**: `pcbang/+/entities/+/long_open` (QoS 1)
2. **long_open 핸들러**: AlertEvent 생성 + WS 브로드캐스트
3. **중복 방지**: 동일 entity에 미확인 장시간개방 알림이 있으면 새 AlertEvent 생성 안 함
4. **EntityType.threshold_minutes**: DB 컬럼 추가 (출입문=10, 냉장고=5, 카운터=None, 기타=None)
5. **monitored_entities payload 변경**: `string[]` → `object[]`

### monitored_entities 새 형식

Backend가 이미 새 형식으로 발행하고 있습니다:

```json
{
  "store_id": 30584,
  "entities": [
    {"ha_entity_id": "binary_sensor.door_entrance", "threshold_minutes": 10},
    {"ha_entity_id": "binary_sensor.fridge_01",     "threshold_minutes": 5},
    {"ha_entity_id": "switch.relay_01",              "threshold_minutes": null}
  ],
  "timestamp": "..."
}
```

**`threshold_minutes: null`** → 장시간개방 감지 비활성 (switch 등)

---

## 통합 테스트 요청

Edge가 구현 완료 후 아래 시나리오로 테스트 부탁드립니다:

1. `monitored_entities` 수신 → `threshold_minutes` 파싱하여 asyncio 타이머 설정 확인
2. binary_sensor `on` 전환 → threshold_minutes 경과 → `long_open` MQTT 발행
3. Backend에서 AlertEvent 생성 → Frontend WS로 `alert.new` 수신 확인
4. 중복 방지: 미확인 알림 있는 상태에서 재발행 시 새 AlertEvent 생성 안 됨 확인

테스트 완료 후 `shared/to_orchestrator/`에 결과 리포트 부탁드립니다.

---

## MQTT 토픽 최종 스펙

```
Topic:   pcbang/{store_id}/entities/{ha_entity_id}/long_open
QoS:     1
Retain:  false
Payload:
{
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
```
