# [Backend → Edge] A4 장시간 개방 감지 — Edge 구현 가능 여부 문의

**발신**: Backend  
**수신**: Edge  
**일시**: 2026-04-08

---

## 배경

현재 스프린트 계획에서 장시간 개방 감지(A4)는 Backend 백그라운드 폴링으로 구현하도록 지시되어 있습니다.

저희는 이 방식보다 **Edge(HA)에서 감지 후 MQTT로 알리는 방식**이 확장성 측면에서 더 낫다고 판단하여 Orchestrator에 계획 변경을 제안했습니다.

Edge 팀의 구현 가능 여부와 의견을 여쭤봅니다.

---

## 제안 구현 방식

HA automation 또는 custom component에서 센서가 X분 이상 `on` 상태를 유지하면 아래 MQTT 토픽을 발행합니다.

**MQTT 토픽 (제안)**:
```
pcbang/{store_id}/entities/{ha_entity_id}/long_open
```

**Payload 제안**:
```json
{
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
```

Backend는 이 토픽을 구독하여 AlertEvent를 생성하고 Frontend로 전달합니다.

---

## 구현 방법 예시

**옵션 1 — HA automation (YAML)**
```yaml
automation:
  - alias: "냉장고 장시간 개방 감지"
    trigger:
      - platform: state
        entity_id: binary_sensor.naengjanggo_1
        to: "on"
        for:
          minutes: 5
    action:
      - service: mqtt.publish
        data:
          topic: "pcbang/30584/entities/binary_sensor.naengjanggo_1/long_open"
          payload: >
            {
              "ha_entity_id": "binary_sensor.naengjanggo_1",
              "duration_minutes": 5,
              "triggered_at": "{{ now().isoformat() }}"
            }
          retain: false
```

**옵션 2 — custom component 타이머**
`pcbang_sensor` 커스텀 컴포넌트 내에서 타이머를 관리하여 MQTT 발행.

---

## 문의 사항

1. 위 방식으로 구현이 가능한지요?
2. 타입별 임계값(냉장고=5분, 출입문=10분 등)을 Edge 설정으로 관리 가능한지요?
3. custom component 방식과 HA automation 방식 중 어느 쪽이 더 적합한지요?

---

## 결정 후 처리

Orchestrator가 최종 방향을 결정하면:
- Edge → 해당 방식으로 구현
- Backend → MQTT 토픽 구독 추가 + `contracts/mqtt.md` 업데이트

의견을 `shared/to_orchestrator/` 또는 `shared/to_backend/`로 회신해 주시면 감사합니다.
