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
| MQTT 포트 | 8883 (TLS, 운영) / 1883 (평문, 로컬 개발) | Edge는 수신한 mqtt_port가 1883이면 TLS 미사용 |

## Heartbeat 주기

- Edge: **30초마다** `pcbang/{store_id}/status` 에 `status: "online"` publish
- Backend: 마지막 heartbeat 수신 후 **90초 초과** 시 해당 매장 `offline` 처리 → WS `store.status` 브로드캐스트
- 재연결 직후 즉시 heartbeat publish (30초 대기 없이)

---

## MQTT Topic 전체 구조

```
# Edge → Backend
pcbang/{store_id}/status                              # Heartbeat (retain=true) + LWT (retain=false)
pcbang/{store_id}/entities/{ha_entity_id}/state       # 모니터링 대상 entity 상태 변화
pcbang/{store_id}/response/entities                   # 전체 entity 목록 응답

# Backend → Edge
pcbang/{store_id}/query/entities                      # 전체 entity 목록 요청
pcbang/{store_id}/config/monitored_entities           # 모니터링할 entity 목록 전달
pcbang/{store_id}/relays/{ha_entity_id}/set           # 릴레이 제어 명령
  ※ Edge는 pcbang/{store_id}/relays/# (와일드카드)로 구독

pcbang/{store_id}/entities/{ha_entity_id}/long_open   # 장시간 개방 감지 (Edge → Backend) [A4 승인됨]
```

---

## 매장 등록 Provisioning

```
1. HA에 Custom Component 설치
2. HA 통합 설정 페이지(Config Flow)에서 store_id(정수)와 backend_url 입력
3. Component → Backend 등록 요청:
   POST {backend_url}/api/v1/edge/register
   { "store_id": 30584, "secret_key": "change-me-shared-with-edge-component" }
4. Backend: secret_key 검증 → MQTT 계정 생성 → 응답
   { "success": true, "data": { "mqtt_host": "pcbang-mosquitto", "mqtt_port": 8883, "username": "30584", "password": "..." } }
5. Component: data 필드 언래핑 후 credentials로 Mosquitto 연결
6. Backend: pcbang/{store_id}/config/monitored_entities 발행 (초기값 빈 목록)
```
> `secret_key`: Custom Component 소스에 하드코딩된 공유 시크릿. Backend와 동일한 값 사용.
> `store_id`는 정수형(int). Backend가 정수로 관리하므로 문자열 전송 시 등록 실패.
> Config Flow에서 `backend_url`을 입력받음 (기본값: `http://host.docker.internal:8080`).
> Backend 응답은 `{"success": true, "data": {...}}` 형식 — Component가 `data` 필드 언래핑.
> Dashboard에서 매장 entity 선택 완료 시 monitored_entities 목록 업데이트 발행.

---

## Payloads

### Edge Heartbeat
**Topic**: `pcbang/{store_id}/status` | **retain: true** | QoS: 1
```json
{ "store_id": 30584, "status": "online", "go2rtc_url": "http://1.2.3.4:1984", "timestamp": "..." }
```

### LWT (Last Will Testament)
**Topic**: `pcbang/{store_id}/status` | **retain: false** | QoS: 1
```json
{ "store_id": 30584, "status": "offline", "timestamp": "" }
```
> `will_set(retain=False)`으로 등록. Mosquitto가 비정상 연결 끊김 시 자동 발행.
> Heartbeat(online)은 retain=true, LWT(offline)은 retain=false — 의도적 구분.

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
> Backend: request_id로 응답 매칭, 10초 timeout 후 에러 반환.
>
> **확정된 entity 제외 목록 (2026-04-08 확정):**
>
> Domain 기준 제외: `sun`, `zone`, `person`, `device_tracker`, `automation`, `script`,
> `scene`, `input_boolean`, `input_number`, `input_select`, `input_text`, `input_datetime`,
> `timer`, `counter`, `persistent_notification`, `system_log`, `update`, `tts`, `todo`,
> `weather`, `conversation`
>
> Platform 기준 제외 (entity registry): `pcbang_sensor`(자체 컴포넌트 entity),
> `backup`, `sun`(sensor.sun_next_* 등), `local_ip`, `dnsip`

### 모니터링 대상 entity 목록
**Topic**: `pcbang/{store_id}/config/monitored_entities` | retain: true
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
> `entities`: `object[]` 형식. `ha_entity_id` + `threshold_minutes` 포함.
> `threshold_minutes: null` — 장시간개방 감지 비활성 (switch 등).
> Backend가 Dashboard에서 entity 선택 저장 시 즉시 발행.
> retain=true → 재연결 시 자동으로 최신 목록 수신.
> Edge는 이 목록을 캐시하고, **목록에 있는 entity의 state_changed만 publish**.
> Edge는 `threshold_minutes` 값을 asyncio 타이머에 적용하여 장시간 개방 감지.

### Entity 상태 변화 (모니터링 대상만)
**Topic**: `pcbang/{store_id}/entities/{ha_entity_id}/state` | retain: true | QoS: 1
```json
{ "store_id": 30584, "ha_entity_id": "binary_sensor.door_entrance", "state": "open", "timestamp": "..." }
```

### 릴레이 제어
**Command** (Backend → Edge): `pcbang/{store_id}/relays/{ha_entity_id}/set` | retain: false | QoS: 1
```json
{ "command": "on | off", "requested_by": "admin1", "timestamp": "..." }
```
> Edge는 `pcbang/{store_id}/relays/#` 와일드카드로 구독.
> `ha_entity_id`의 `.`은 MQTT 토픽에 그대로 유지 (예: `relays/switch.relay_01/set`).
> Component 수신 후 `hass.services.async_call("switch", "turn_on/off", ...)` 호출.
> 결과는 `entities/{ha_entity_id}/state` topic으로 자동 반영.

### 장시간 개방 감지 (A4 — 승인됨)
**Topic**: `pcbang/{store_id}/entities/{ha_entity_id}/long_open` | retain: false | QoS: 1
```json
{
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
```
> Edge `coordinator.py` asyncio 타이머 방식으로 구현 (Custom Component 내).
> `threshold_minutes`는 `monitored_entities` payload의 object[]에서 수신.
> Entity가 `on` 상태 전환 시 타이머 시작, `off` 전환 시 타이머 취소.
> Backend는 이 토픽 구독 후 AlertEvent(`type_name='장시간개방'`) 생성 → WS 브로드캐스트.
> 중복 방지: 동일 entity에 미확인 장시간개방 알림 있으면 새 AlertEvent 생성 안 함.

---

## HA Custom Component 동작 요약

| 시점 | 동작 |
|---|---|
| 최초 설치 (Config Flow) | store_id(정수) + backend_url 입력 → Backend 등록 요청 → MQTT credentials 수신 저장 |
| 시작 시 | Mosquitto 연결 (port!=1883이면 TLS), LWT 등록(retain=false), 즉시 heartbeat publish(retain=true), `config/monitored_entities` + `query/entities` + `relays/#` 구독 |
| `config/monitored_entities` 수신 | 모니터링 entity 목록 메모리 캐시 업데이트 (retain=true이므로 재연결 시 자동 재수신) |
| HA state_changed 이벤트 | 모니터링 목록에 있는 경우에만 MQTT publish |
| `query/entities` 수신 | 확정된 제외 목록 필터링 후 전체 entity 목록 응답 |
| `relays/…/set` 수신 | `asyncio.run_coroutine_threadsafe`로 switch 서비스 비동기 호출 (paho 스레드 안전) |
| 연결 끊김 | LWT 자동 발행 → Backend offline 감지. 지수 백오프(5s→60s)로 재연결 |
| 재연결 성공 | 즉시 heartbeat publish. `config/monitored_entities`는 retain=true이므로 Mosquitto가 자동 재전송 → 캐시 자동 복원 |
| MQTT 연결 상태 확인 | HA 내 MQTT 연결 상태 sensor entity로 확인 가능 (entity 조회 응답에는 미포함 — pcbang_sensor platform 제외) |
