# MQTT 통신 계약

## 전제
- HA Custom Component는 **paho-mqtt**를 직접 사용하여 중앙 Mosquitto에 연결
- HA 내장 MQTT integration 사용 안 함 → topic 구조 완전 제어
- 모든 메시지 JSON, QoS 1, retain 여부 항목별 명시
- Edge는 Backend로부터 받은 **monitored entity 목록에 있는 entity의 상태 변화만** 발행

## MQTT 연결 파라미터 (paho-mqtt)

| 파라미터 | 값 | 설명 |
|---|---|---|
| keepalive | 60초 | Mosquitto가 60초 이상 무응답 시 연결 끊음 |
| clean_session | **Edge: False** / **Backend: True** | Edge는 재연결 시 QoS 1 미수신 메시지 유지. Backend는 uvicorn reload 시 client_id 충돌 방지를 위해 True 사용 (client_id에 PID 포함: `pcbang-backend-{pid}`) |
| reconnect_on_failure | True | 자동 재연결 활성화 |
| reconnect_delay | 5초 (초기) / 최대 60초 (지수 백오프) | 재연결 간격 |
| LWT QoS | 1 | Mosquitto가 offline 메시지 발행 시 QoS |

## Heartbeat 주기

- Edge: **30초마다** `pcbang/{store_id}/status` 에 `status: "online"` publish
- Backend: 마지막 heartbeat 수신 후 **90초 초과** 시 해당 매장 `offline` 처리 → WS `store.status` 브로드캐스트
- 재연결 직후 즉시 heartbeat publish (30초 대기 없이)

---

## MQTT Topic 전체 구조

```
# Edge → Backend
pcbang/{store_id}/status                              # Heartbeat + LWT
pcbang/{store_id}/entities/{ha_entity_id}/state       # 모니터링 대상 entity 상태 변화
pcbang/{store_id}/response/entities                   # 전체 entity 목록 응답

# Backend → Edge
pcbang/{store_id}/query/entities                      # 전체 entity 목록 요청
pcbang/{store_id}/config/monitored_entities           # 모니터링할 entity 목록 전달
pcbang/{store_id}/relays/{ha_entity_id}/set           # 릴레이 제어 명령
```

---

## 매장 등록 Provisioning

```
1. HA에 Custom Component 설치
2. HA 통합 설정 페이지(Config Flow)에서 store_id(관제서버 매장 번호, 숫자) 입력
3. Component → Backend 등록 요청:
   POST https://{backend}/api/v1/edge/register
   { "store_id": 30584, "secret_key": "<COMPONENT_HARDCODED_KEY>" }
4. Backend: secret_key 검증 → MQTT 계정 생성 → 응답
   { "mqtt_host": "...", "mqtt_port": 8883, "username": "30584", "password": "..." }
5. Component: 수신한 credentials로 Mosquitto 연결
6. Backend: pcbang/{store_id}/config/monitored_entities 발행 (초기값 빈 목록)
```
> `COMPONENT_HARDCODED_KEY`: Custom Component 소스에 하드코딩된 공유 시크릿.
> Dashboard에서 매장 entity 선택 완료 시 monitored_entities 목록 업데이트 발행.

---

## Payloads

### Edge Heartbeat (+ LWT)
**Topic**: `pcbang/{store_id}/status` | retain: true
```json
{ "store_id": 30584, "status": "online", "go2rtc_url": "http://1.2.3.4:1984", "timestamp": "..." }
```
> LWT: `{ "store_id": 30584, "status": "offline", "timestamp": "" }` (Mosquitto가 자동 발행)

### Entity 전체 목록 요청 / 응답
**Request** (Backend → Edge): `pcbang/{store_id}/query/entities` | retain: false
```json
{ "request_id": "uuid" }
```

**Response** (Edge → Backend): `pcbang/{store_id}/response/entities` | retain: false
```json
{
  "request_id": "uuid",
  "store_id": 30584,
  "entities": [
    { "ha_entity_id": "binary_sensor.door_entrance", "entity_kind": "sensor", "domain": "binary_sensor", "current_state": "closed", "friendly_name": "Door Entrance" },
    { "ha_entity_id": "switch.relay_01",             "entity_kind": "switch", "domain": "switch",        "current_state": "off",    "friendly_name": "Relay 01" }
  ],
  "timestamp": "..."
}
```
> Backend: request_id로 응답 매칭, 10초 timeout 후 에러 반환
> HA 기본 내장 entity(sun, zone, person, device_tracker 등) 제외하여 전송.
> 제외 목록은 Edge 개발 시 테스트 후 사용자와 협의하여 확정.

### 모니터링 대상 entity 목록
**Topic**: `pcbang/{store_id}/config/monitored_entities` | retain: true
```json
{
  "store_id": 30584,
  "entities": ["binary_sensor.door_entrance", "binary_sensor.fridge_01", "switch.relay_01"],
  "timestamp": "..."
}
```
> Backend가 Dashboard에서 entity 선택 저장 시 즉시 발행.
> retain=true → 재연결 시 자동으로 최신 목록 수신.
> Edge는 이 목록을 캐시하고, **목록에 있는 entity의 state_changed만 publish**.

### Entity 상태 변화 (모니터링 대상만)
**Topic**: `pcbang/{store_id}/entities/{ha_entity_id}/state` | retain: true
```json
{ "store_id": 30584, "ha_entity_id": "binary_sensor.door_entrance", "state": "on", "timestamp": "..." }
```
> HA binary_sensor 상태값은 `"on"` / `"off"` 사용 (`"open"` / `"closed"` 아님)

### 릴레이 제어
**Command** (Backend → Edge): `pcbang/{store_id}/relays/{ha_entity_id}/set` | retain: false
```json
{ "command": "on | off", "requested_by": "admin1", "timestamp": "..." }
```
> Component 수신 후 `hass.services.call("switch", "turn_on/off", ...)` 호출
> 결과는 `entities/{ha_entity_id}/state` topic으로 자동 반영

---

## HA Custom Component 동작 요약

| 시점 | 동작 |
|---|---|
| 최초 설치 (Config Flow) | store_id 입력 → Backend 등록 요청 → MQTT credentials 수신 저장 |
| 시작 시 | Mosquitto 연결, LWT 등록, 즉시 heartbeat publish, `config/monitored_entities` 구독 |
| `config/monitored_entities` 수신 | 모니터링 entity 목록 메모리 캐시 업데이트 (retain=true이므로 재연결 시 자동 재수신) |
| HA state_changed 이벤트 | 모니터링 목록에 있는 경우에만 MQTT publish |
| `query/entities` 수신 | HA 기본 entity 제외한 전체 목록 응답 |
| `relays/.../set` 수신 | hass.services.call()로 switch 제어 |
| 연결 끊김 | LWT 자동 발행 → Backend offline 감지 |
| 재연결 성공 | 즉시 heartbeat publish. `config/monitored_entities`는 retain=true이므로 Mosquitto가 자동 재전송 → 캐시 자동 복원 |
