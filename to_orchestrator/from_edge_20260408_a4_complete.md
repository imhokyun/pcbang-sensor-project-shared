# [Edge → Orchestrator] A4-Edge 구현 완료 보고

**발신**: Edge  
**수신**: Orchestrator  
**일시**: 2026-04-08

---

## 결론

**A4-Edge 구현 완료. 테스트 47/47 통과.**

---

## 구현 내용

**파일**: `edge/homeassistant/custom_components/pcbang_sensor/coordinator.py`

| 항목 | 내용 |
|------|------|
| 타이머 필드 | `_entity_thresholds: dict[str, int]`, `_long_open_timers: dict[str, asyncio.Task]` |
| payload 파싱 | object[] / string[] 자동 감지 — 하위 호환 유지 |
| 타이머 시작 | `on` 전환 + threshold 있는 경우 `hass.loop.create_task()` |
| 타이머 취소 | `off` 전환, monitored_entities 변경, teardown 시 |
| 만료 처리 | 현재 상태 재확인 후 publish (이중 안전장치) |
| 정리 | teardown 시 전체 타이머 cancel + clear |

---

## MQTT 토픽

```
Topic:   pcbang/{store_id}/entities/{ha_entity_id}/long_open
Payload: {
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
QoS: 1, retain: false
```

---

## 테스트 결과

| 테스트 | 결과 |
|--------|------|
| object[] 형식 파싱 | ✅ |
| string[] 구형 하위 호환 | ✅ |
| on 전환 시 타이머 시작 | ✅ |
| off 전환 시 타이머 취소 | ✅ |
| threshold 없으면 타이머 미시작 | ✅ |
| 만료 시 long_open publish | ✅ |
| 만료 시 entity off이면 미발행 | ✅ |
| monitored_entities 변경 시 타이머 리셋 | ✅ |
| 전체 (기존 포함) | **47 / 47** |

---

## 대기 사항

Backend A4-BE (MQTT 구독 + AlertEvent 생성) 완료 통보 수신 후 통합 테스트 진행 예정:

1. HA에서 `binary_sensor` 수동으로 `on` 설정
2. `threshold_minutes` 경과 확인
3. Backend AlertEvent 생성 확인
4. Frontend "장시간개방" 알림 수신 확인

결과는 `shared/to_orchestrator/from_edge_20260408_a4_integration_test.md`에 보고 예정.
