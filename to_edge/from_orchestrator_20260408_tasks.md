# Edge 작업 지시

**발신**: Orchestrator | **날짜**: 2026-04-08 | **브랜치**: dev/edge  
**전체 그림**: `shared/new_request_20260408.md` 먼저 읽어볼 것

---

## 담당 작업 요약

| ID | 작업 | 의존성 | 우선순위 |
|----|------|--------|---------|
| A4-Edge | HA 장시간 개방 자동화 보완 | Backend A4-BE 완료 후 | **선택(Optional)** |

이번 스프린트에서 Edge의 필수 작업은 없습니다.  
A4-Edge는 Backend의 백그라운드 태스크 방식만으로도 동작하지만,  
Edge에서도 보완하면 더 빠른 감지가 가능합니다.

---

## A4-Edge — HA 장시간 개방 자동화 (선택)

### 배경

Backend가 1분 주기로 DB를 체크하여 장시간 개방 알림을 생성합니다.  
Edge(HA)에서 추가로 자동화를 구성하면:
- 더 빠른 감지 (HA 타이머 기반, 초 단위 정밀도)
- Backend DB 폴링 부하 경감

### 구현 옵션

**옵션 1: HA 자동화 (automation) — YAML**
```yaml
# configuration.yaml 또는 automations.yaml
automation:
  - alias: "센서 장시간 개방 감지"
    trigger:
      - platform: state
        entity_id:
          - binary_sensor.naengjanggo_1
          - binary_sensor.humun_senseo
          - binary_sensor.jeongmun_senseo
          # ... 등록된 모든 binary_sensor
        to: "on"
        for:
          minutes: 5  # 타입별로 다르게 하려면 각 센서마다 개별 자동화 필요
    action:
      - service: mqtt.publish
        data:
          topic: "pcbang/store/{{ trigger.to_state.attributes.store_id }}/long_open"
          payload: >
            {
              "entity_id": "{{ trigger.entity_id }}",
              "duration_minutes": 5,
              "triggered_at": "{{ now().isoformat() }}"
            }
          retain: false
```

**옵션 2: Custom Component에서 처리**  
`pcbang_sensor` 커스텀 컴포넌트 내에서 타이머 관리 후 MQTT 발행.

### MQTT 토픽 스펙 (Backend와 협의 필요)

```
Topic:   pcbang/store/{store_id}/long_open
Payload: {
  "entity_id": "binary_sensor.xxx",
  "ha_entity_id": "binary_sensor.xxx",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+09:00"
}
```

이 토픽을 Backend가 수신하도록 하려면 `shared/to_backend/` 에 MQTT 계약 변경 통보 필요.

### 권장사항

Backend의 DB 폴링 방식만으로 충분히 동작합니다.  
Edge 자동화는 나중에 추가 최적화로 진행해도 됩니다.  
**이번 스프린트는 Backend 완료 후 테스트에만 참여** 해주세요.

---

## 테스트 협조 요청

Backend A4-BE 작업 완료 후:
1. 실제 HA에서 `binary_sensor` 를 수동으로 `on` 상태로 만든 후
2. threshold_minutes 경과 후 Frontend 알림 화면에 "장시간개방" 알림이 뜨는지 확인
3. 결과를 `shared/to_orchestrator/` 에 리포트 부탁드립니다.
