# [Frontend → Backend] Entity 상태 변화 → Alert 생성 흐름 확인 요청

**일자**: 2026-04-07  
**발신**: Frontend 팀  
**수신**: Backend 팀

---

## 현상

Edge에서 entity 상태 변화 MQTT가 정상 발행되고 있으나, 메인 관제 화면에 알림이 수신되지 않음.

### Edge MQTT 발행 확인 (정상)

```
topic   : pcbang/30584/entities/binary_sensor.jeongmun_senseo/state
retain  : True
payload : {
  "store_id": 30584,
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "state": "on",
  "timestamp": "2026-04-07T03:43:19.390388+00:00"
}
```

### 프론트엔드 WS 이벤트 처리 (구현 완료)

프론트는 아래 두 이벤트를 정상 처리합니다:

- `alert.new` → 알림 리스트 최상단 추가 + 소리 재생
- `entity.update` → 매장 카드 센서 상태 갱신

---

## 확인 요청 사항

### 1. MQTT → Alert 생성 흐름

아래 흐름이 백엔드에 구현되어 있는지 확인 부탁드립니다:

```
MQTT 수신 (pcbang/{store_id}/entities/{ha_entity_id}/state)
  └→ DB에서 해당 entity 조회 (store_id + ha_entity_id)
       └→ triggers_alert = 1 이면
            └→ alerts 테이블에 INSERT
                 └→ WebSocket으로 alert.new 브로드캐스트
```

### 2. entity.update 브로드캐스트

MQTT 상태 변화 수신 시, `triggers_alert` 여부와 무관하게 아래 이벤트를 WS로 브로드캐스트해주세요:

```json
{
  "type": "entity.update",
  "store_id": 30584,
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "state": "on"
}
```

### 3. alert.new 페이로드 스펙

프론트가 기대하는 `alert.new` 이벤트 형식입니다. 현재 구현과 다르면 알려주세요:

```json
{
  "type": "alert.new",
  "alert_id": 1,
  "store_id": 30584,
  "store_name": "테스트매장",
  "importance": 3,
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "entity_name": "정문 센서",
  "message": "정문 센서 on",
  "is_in_schedule": true,
  "timestamp": "2026-04-07T03:43:19.390388+00:00"
}
```

---

## 현재 테스트 환경

- store_id: 30584
- 등록된 entity: `binary_sensor.jeongmun_senseo` (triggers_alert 설정 여부 확인 필요)
- Edge → MQTT 발행 정상 확인됨
- 백엔드 MQTT 수신 여부 및 Alert 생성 로직 동작 여부 미확인

---

## 요청

1. MQTT 수신 후 alert 생성 및 WS 브로드캐스트 로직 구현 여부 확인
2. 미구현 시 위 스펙대로 구현 요청
3. 구현 완료 후 `shared/to_frontend/` inbox로 완료 통보 부탁드립니다
