# Backend 준비 완료 — Edge 연동 가이드

**발신**: backend | **수신**: edge | **날짜**: 2026-04-07

---

## 현재 실행 중인 서비스

| 서비스 | 주소 | 비고 |
|---|---|---|
| MQTT Broker (Mosquitto) | `localhost:1883` | TLS 없음 (개발 환경) |
| Backend API | `http://localhost:8080/api/v1` | |
| Backend Swagger | `http://localhost:8080/docs` | |

---

## Edge 등록 절차

### 1단계: 매장을 Backend에 먼저 등록 (관리자 UI 또는 curl)

```bash
# 로그인
curl -c /tmp/cookies.txt -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin1","password":"admin1234"}'

# 매장 등록
curl -b /tmp/cookies.txt -X POST http://localhost:8080/api/v1/stores \
  -H "Content-Type: application/json" \
  -d '{"store_id":30584,"name":"강남점","address":"서울 강남구","importance":4}'
```

### 2단계: Edge 컴포넌트에서 등록 요청 (인증 불필요)

```bash
curl -X POST http://localhost:8080/api/v1/edge/register \
  -H "Content-Type: application/json" \
  -d '{"store_id":30584,"secret_key":"change-me-shared-with-edge-component"}'
```

응답:
```json
{
  "success": true,
  "data": {
    "mqtt_host": "localhost",
    "mqtt_port": 1883,
    "username": "30584",
    "password": "<generated>"
  }
}
```

> `secret_key`는 Edge Custom Component에 하드코딩할 공유 시크릿. 현재 개발용 값: `change-me-shared-with-edge-component`

---

## MQTT 연결 파라미터

```python
client = mqtt.Client(client_id="30584", clean_session=False, protocol=mqtt.MQTTv311)
client.username_pw_set(username="30584", password="<register 응답값>")
client.connect("localhost", 1883, keepalive=60)
```

---

## 발행해야 할 토픽

### Heartbeat (30초마다 + 연결 직후 즉시)
```
Topic : pcbang/{store_id}/status
QoS   : 1
Retain: true
Payload:
{
  "store_id": 30584,
  "status": "online",
  "go2rtc_url": "http://<edge_ip>:1984",
  "timestamp": "2026-04-07T10:00:00Z"
}
```
> 90초 미수신 시 Backend가 offline 전환 → WS `store.status` 브로드캐스트

### LWT (Mosquitto 자동 발행)
```
Topic : pcbang/{store_id}/status
Payload: { "store_id": "store_001", "status": "offline", "timestamp": "" }
```

### Entity 상태 변화 (모니터링 대상만)
```
Topic : pcbang/{store_id}/entities/{ha_entity_id}/state
QoS   : 1
Retain: true
Payload:
{
  "store_id": 30584,
  "ha_entity_id": "binary_sensor.door_entrance",
  "state": "open",
  "timestamp": "2026-04-07T10:00:01Z"
}
```

### Entity 전체 목록 응답 (query 수신 시)
```
Topic : pcbang/{store_id}/response/entities
QoS   : 1
Retain: false
Payload:
{
  "request_id": "<Backend이 보낸 uuid>",
  "store_id": 30584,
  "entities": [
    {
      "ha_entity_id": "binary_sensor.door_entrance",
      "entity_kind": "sensor",
      "domain": "binary_sensor",
      "current_state": "closed",
      "friendly_name": "출입구 도어"
    },
    {
      "ha_entity_id": "switch.relay_01",
      "entity_kind": "switch",
      "domain": "switch",
      "current_state": "off",
      "friendly_name": "릴레이 1"
    }
  ],
  "timestamp": "2026-04-07T10:00:00Z"
}
```

---

## 구독해야 할 토픽

```
pcbang/{store_id}/query/entities           → 전체 entity 목록 요청 수신 시 응답 발행
pcbang/{store_id}/config/monitored_entities → 모니터링 대상 entity 목록 (retain=true, 재연결 시 자동 재수신)
pcbang/{store_id}/relays/{ha_entity_id}/set → 릴레이 제어 명령
```

### `config/monitored_entities` 수신 예시
```json
{
  "store_id": 30584,
  "entities": ["binary_sensor.door_entrance", "binary_sensor.fridge_01"],
  "timestamp": "..."
}
```
> 이 목록에 있는 entity의 state_changed 이벤트만 발행

### `relays/.../set` 수신 예시
```json
{ "command": "on", "requested_by": "admin1", "timestamp": "..." }
```
> 수신 후 `hass.services.call("switch", "turn_on", ...)` 호출

---

## 빠른 연동 테스트 (MQTT 클라이언트로 직접 확인)

```bash
# Mosquitto 클라이언트 설치 후 heartbeat 발행 테스트
mosquitto_pub -h localhost -p 1883 \
  -u 30584 -P "<password>" \
  -t "pcbang/30584/status" -r -q 1 \
  -m '{"store_id":30584,"status":"online","go2rtc_url":"http://192.168.1.100:1984","timestamp":"2026-04-07T10:00:00Z"}'
```

Backend가 수신하면 해당 매장 `status`가 `online`으로 바뀌고 WS로 `store.status` 이벤트가 브로드캐스트됩니다.
