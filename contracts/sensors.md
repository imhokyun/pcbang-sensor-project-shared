# MQTT 통신 계약 정의

## 전제
- HA Custom Component는 **paho-mqtt**를 직접 사용하여 중앙 Mosquitto에 연결
- HA 내장 MQTT integration 사용 안 함 → topic 구조 완전 제어
- 모든 메시지는 JSON, QoS 1, retain 여부는 항목별 명시

---

## MQTT Topic 전체 구조

```
# Edge → Backend (publish)
pcbang/{store_id}/status                              # Heartbeat
pcbang/{store_id}/entities/{ha_entity_id}/state       # Entity 상태 변화
pcbang/{store_id}/response/entities                   # Entity 목록 응답

# Backend → Edge (publish)
pcbang/{store_id}/query/entities                      # Entity 목록 요청
pcbang/{store_id}/relays/{ha_entity_id}/set           # 릴레이 제어 명령
```

---

## Payloads

### Edge Heartbeat
**Topic**: `pcbang/{store_id}/status` | retain: true
```json
{
  "store_id": "store_001",
  "status": "online",
  "go2rtc_url": "http://1.2.3.4:1984",
  "timestamp": "2026-04-06T22:00:00Z"
}
```
> LWT(Last Will Testament)로 `status: "offline"` 설정하여 연결 끊김 자동 감지

### Entity 상태 변화
**Topic**: `pcbang/{store_id}/entities/{ha_entity_id}/state` | retain: true
```json
{
  "store_id": "store_001",
  "ha_entity_id": "binary_sensor.door_entrance",
  "state": "open | closed | on | off | <숫자>",
  "timestamp": "2026-04-06T22:05:00Z"
}
```
> Custom Component는 HA state_changed 이벤트 수신 시 즉시 publish

### Entity 목록 요청 / 응답
**Request** (Backend → Edge): `pcbang/{store_id}/query/entities` | retain: false
```json
{ "request_id": "uuid" }
```

**Response** (Edge → Backend): `pcbang/{store_id}/response/entities` | retain: false
```json
{
  "request_id": "uuid",
  "store_id": "store_001",
  "entities": [
    {
      "ha_entity_id": "binary_sensor.door_entrance",
      "entity_kind": "sensor",
      "domain": "binary_sensor",
      "current_state": "closed",
      "friendly_name": "Door Entrance"
    },
    {
      "ha_entity_id": "switch.relay_01",
      "entity_kind": "switch",
      "domain": "switch",
      "current_state": "off",
      "friendly_name": "Relay 01"
    }
  ],
  "timestamp": "2026-04-06T22:00:00Z"
}
```
> Backend는 request_id로 응답 매칭, 10초 timeout 후 에러 반환

### 릴레이 제어
**Command** (Backend → Edge): `pcbang/{store_id}/relays/{ha_entity_id}/set` | retain: false
```json
{
  "command": "on | off",
  "requested_by": "admin1",
  "timestamp": "2026-04-06T22:10:00Z"
}
```
> Custom Component 수신 후 `hass.services.call("switch", "turn_on/off", ...)` 호출
> 실행 결과는 `entities/{ha_entity_id}/state` topic으로 자동 반영됨

---

## HA Custom Component 동작 요약

| 시점 | 동작 |
|---|---|
| 시작 시 | 중앙 Mosquitto에 paho-mqtt로 연결, LWT 등록, heartbeat publish |
| state_changed 이벤트 | 해당 entity state를 MQTT publish |
| `query/entities` 수신 | 전체 등록 entity 목록 응답 publish |
| `relays/.../set` 수신 | HA service call로 switch 제어 |
| 연결 끊김 | LWT 자동 발행 → Backend가 offline 감지 |
