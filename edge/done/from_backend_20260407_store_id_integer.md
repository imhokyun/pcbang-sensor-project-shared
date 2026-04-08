# store_id 타입 변경 알림 — TEXT → INTEGER

**발신**: backend | **수신**: edge | **날짜**: 2026-04-07

---

## 변경 내용

`store_id` 필드가 **TEXT("store_001")에서 INTEGER(30584)로 변경**되었습니다.

외부 관제관리서버의 숫자형 `storeNo`(예: 30584, 30585)와 통일합니다.

---

## Edge 영향 범위

### 1. HA Config Flow에서 입력값

사용자가 HA Custom Component 설정 시 입력하는 store_id는 **숫자**입니다:
```
Store ID를 입력하세요: 30584
```
→ Component 내부에서 integer로 처리

### 2. Backend 등록 요청

```python
POST /api/v1/edge/register
{
  "store_id": 30584,   # integer (기존: "store_001" string)
  "secret_key": "change-me-shared-with-edge-component"
}
```

응답:
```json
{
  "success": true,
  "data": {
    "mqtt_host": "...",
    "mqtt_port": 1883,
    "username": "30584",   # MQTT username은 여전히 string
    "password": "..."
  }
}
```

### 3. MQTT 연결

MQTT client_id와 username은 **문자열** `"30584"` (숫자를 문자열로 변환)입니다:
```python
client.username_pw_set(username="30584", password="<password>")
client.connect(host, port)
```

### 4. MQTT 토픽

토픽 구조는 동일, store_id 부분만 숫자 문자열로:
```
pcbang/30584/status
pcbang/30584/entities/{ha_entity_id}/state
pcbang/30584/response/entities
pcbang/30584/config/monitored_entities
pcbang/30584/query/entities
pcbang/30584/relays/{ha_entity_id}/set
```

### 5. Payload의 store_id

MQTT payload 내 `store_id`는 **integer**:
```json
{
  "store_id": 30584,
  "status": "online",
  "go2rtc_url": "http://192.168.x.x:1984",
  "timestamp": "2026-04-07T10:00:00Z"
}
```

---

## 계약 문서 업데이트

- `shared/contracts/mqtt.md` 반영 완료
- `shared/contracts/api.md` 반영 완료
