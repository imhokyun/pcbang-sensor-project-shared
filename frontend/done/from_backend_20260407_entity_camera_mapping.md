# [Backend → Frontend] 센서-카메라 채널 연결 기능 추가

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

---

## 개요

센서(entity)와 카메라 채널을 1:1로 연결할 수 있는 `camera_channel` 필드가 추가됐습니다.  
alert 발생 시 해당 카메라의 `stream_url`이 자동으로 포함되어 전달됩니다.

---

## 변경된 API

### Entity 응답에 `camera_channel` 추가

**GET /api/v1/stores/{store_id}/entities**
```json
{
  "id": 21,
  "store_id": 30584,
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "custom_name": "정문 센서",
  "triggers_alert": 1,
  "camera_channel": 1,   ← 신규 (null이면 미연결)
  "current_state": "off"
}
```

### Entity 생성/수정 시 camera_channel 설정 가능

**POST /api/v1/stores/{store_id}/entities**
```json
{
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "entity_kind": "sensor",
  "custom_name": "정문 센서",
  "type_id": 1,
  "triggers_alert": 1,
  "camera_channel": 1
}
```

**PUT /api/v1/stores/{store_id}/entities/{ha_entity_id}**
```json
{ "camera_channel": 1 }      // 연결
{ "camera_channel": null }   // 연결 해제
```

---

## Alert 발생 시 stream_url 자동 포함

`camera_channel`이 설정된 센서에서 alert가 발생하면, `alert.new` WS 이벤트와 `GET /api/v1/alerts` 응답에 해당 카메라의 `stream_url`이 자동으로 포함됩니다.

```json
{
  "type": "alert.new",
  "alert_id": 1,
  "store_id": 30584,
  "store_name": "강남점",
  "ha_entity_id": "binary_sensor.jeongmun_senseo",
  "entity_name": "정문 센서",
  "message": "정문 센서 on",
  "stream_url": "http://localhost:1984/stream.html?src=ch1_sub",  ← 자동 세팅
  "is_in_schedule": true,
  "timestamp": "..."
}
```

`stream_url`이 null이면 해당 센서에 카메라가 연결되지 않은 것입니다.

---

## 프론트엔드 구현 가이드

### 매장 설정 화면 (센서-카메라 매핑)

1. `GET /stores/{store_id}/cameras` → 카메라 채널 목록 조회
2. `GET /stores/{store_id}/entities` → 센서 목록 조회
3. 센서별로 카메라 채널 선택 드롭다운 표시 (`camera_channel` 필드)
4. 선택 시 `PUT /stores/{store_id}/entities/{ha_entity_id}` → `{ "camera_channel": N }` 전송

### Alert 팝업

- `alert.new.stream_url`이 있으면 → iframe으로 즉시 카메라 영상 표시
- `stream_url`이 null이면 → 카메라 영상 없이 알림만 표시
