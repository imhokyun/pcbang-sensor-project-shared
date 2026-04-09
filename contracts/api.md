# API 계약 정의

Base URL: `http://backend:8080/api/v1`

---

## 인증 방식

- **방식**: HttpOnly 쿠키 세션
- `POST /auth/login` 성공 시 → `Set-Cookie: session_id=<UUID>; HttpOnly; SameSite=Lax; Path=/`
- 이후 모든 REST 요청 및 **WebSocket 연결**에 쿠키 자동 포함
- 세션 만료 시 401 응답 → 클라이언트가 `/login`으로 리다이렉트
- 세션 TTL: 24시간 (rolling)

---

## REST Endpoints

### Auth
| Method | Path | 설명 |
|---|---|---|
| POST | /auth/login | 로그인 → HttpOnly 쿠키 세션 발급 |
| POST | /auth/logout | 로그아웃 → 세션 삭제 + 쿠키 만료 |

**POST /auth/login request body:**
```json
{ "username": "admin1", "password": "..." }
```
**response:**
```json
{ "success": true, "data": { "username": "admin1", "display_name": "관제 1" } }
```

### Stores
| Method | Path | 설명 |
|---|---|---|
| GET | /stores | 매장 목록 (검색·페이지네이션 지원, B1-BE 완료 후 응답형식 변경) |
| GET | /stores/{store_id} | 매장 상세 |
| POST | /stores | 매장 등록 |
| PUT | /stores/{store_id} | 매장 정보 수정 (name, address, device_sn, importance, force_alert) |
| DELETE | /stores/{store_id} | 매장 삭제 |

**GET /stores query params** (B1-BE 완료 후 적용):
- `q`: 매장명/주소 검색 (ILIKE)
- `page`: 페이지 번호 (기본 1)
- `limit`: 페이지 크기 (기본 100, 최대 100)

**GET /stores response** (B1-BE 완료 후 형식 변경):
```json
{
  "success": true,
  "data": {
    "items": [ /* Store[] */ ],
    "total": 15,
    "page": 1,
    "total_pages": 1
  }
}
```
> B1-BE 완료 전: `"data": Store[]` 형식 유지.
> B1-BE 완료 후: `"data": { items, total, page, total_pages }` 형식으로 변경.

**POST /stores request body:**
```json
{ "store_id": 30584, "name": "강남점", "address": "서울 강남구 ...", "device_sn": "RPI4-XXXX", "importance": 3 }
```
**PUT /stores/{store_id} request body** (부분 수정 가능, 변경할 필드만 포함):
```json
{ "name": "강남점", "address": "...", "device_sn": "...", "importance": 4, "force_alert": null }
```
> `force_alert`: `null`=스케줄 따름, `0`=강제 OFF, `1`=강제 ON

### Cameras
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/cameras | 카메라 채널 목록 |
| POST | /stores/{store_id}/cameras | 카메라 채널 추가 |
| PUT | /stores/{store_id}/cameras/{channel} | 채널 정보 수정 |
| DELETE | /stores/{store_id}/cameras/{channel} | 채널 삭제 |

**카메라 응답 예시** (`GET /stores/30584/cameras`):
```json
{
  "channel": 1,
  "name": "CH1",
  "stream_source": "ch1_sub",
  "stream_url": "http://localhost:1984/stream.html?src=ch1_sub"
}
```
> `stream_url` = `stores.go2rtc_url` + `/stream.html?src=` + `stream_source` (Backend가 조합하여 반환)  
> go2rtc 소스 설정(RTSP 연결 등)은 Edge 팀 담당. Backend는 소스명(`stream_source`)만 저장.

### HA Entities
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/ha/entities | HA에서 전체 entity 목록 조회 (실시간 MQTT query, 10초 timeout) |
| GET | /stores/{store_id}/entities | 등록된 entity 목록 + 현재 상태 |
| POST | /stores/{store_id}/entities | entity 등록 |
| PUT | /stores/{store_id}/entities/{ha_entity_id} | entity 수정 |
| DELETE | /stores/{store_id}/entities/{ha_entity_id} | entity 제거 + monitored_entities 재발행 |

**POST /stores/{store_id}/entities request body:**
```json
{ "ha_entity_id": "binary_sensor.door_01", "entity_kind": "sensor", "custom_name": "출입구 도어", "type_id": 1, "triggers_alert": 1, "camera_channel": 1 }
```
> `camera_channel`: 연결할 카메라 채널 번호 (NULL=미연결). alert 발생 시 해당 채널 stream_url 자동 포함.

**PUT /stores/{store_id}/entities/{ha_entity_id} request body** (부분 수정):
```json
{ "custom_name": "메인 도어", "type_id": 1, "triggers_alert": 1, "camera_channel": 1 }
```

### Entity Types
| Method | Path | 설명 |
|---|---|---|
| GET | /entity-types | 전체 타입 목록 (임계값 없음, 타입 선택용) |
| POST | /entity-types | 사용자 정의 타입 추가 |
| DELETE | /entity-types/{id} | 사용자 정의 타입 삭제 (기본값 삭제 불가) |
| GET | /stores/{store_id}/entity-types | 매장별 타입 목록 + 장시간 개방 임계값 |
| PUT | /stores/{store_id}/entity-types/{type_id} | 매장별 장시간 개방 임계값 수정 |

#### GET /entity-types 응답
```json
{ "id": 1, "name": "출입문", "is_default": 1 }
```

#### GET /stores/{store_id}/entity-types 응답
```json
{ "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 30.0 }
```

#### PUT /stores/{store_id}/entity-types/{type_id} 요청/응답
```json
// Request
{ "threshold_minutes": 30 }   // number | null (null 시 기본값 5분으로 초기화)

// Response
{ "success": true, "data": { "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 30.0 } }
```

### Relays (switch entity 제어)
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/relays | 스위치 entity 목록 + 현재 상태 |
| POST | /stores/{store_id}/relays/{ha_entity_id}/command | on/off 제어 → MQTT publish |

### Monitoring Schedules
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/schedules | 요일별 관제 시간 목록 |
| PUT | /stores/{store_id}/schedules/{day_of_week} | 수동 시간 수정 (is_manual=true) |
| POST | /stores/{store_id}/schedules/sync | 외부 서버에서 즉시 폴링하여 업데이트 |

**PUT /stores/{store_id}/schedules/{day_of_week} request body:**
```json
{ "start_time": "22:00", "end_time": "12:00", "is_active": 1 }
```
> `day_of_week`: 0=월 1=화 2=수 3=목 4=금 5=토 6=일
> `end_time < start_time` → 익일까지 관제 (예: 22:00~12:00 = 익일 정오까지)

### Alerts
| Method | Path | 설명 |
|---|---|---|
| GET | /alerts | 미확인 알림 목록 (전체 매장) |
| POST | /alerts/{alert_id}/acknowledge | 알림 확인 처리 → WS 브로드캐스트 |

### Logs
| Method | Path | 설명 |
|---|---|---|
| GET | /logs | 이벤트 이력 조회 |

**Query params**: `store_id`, `type_name`, `state_from`, `state_to`, `from` (datetime ISO8601), `to` (datetime ISO8601), `page` (기본 1), `limit` (기본 50, 최대 200)

**GET /logs response:**
```json
{
  "success": true,
  "data": {
    "items": [ { "id": 1, "store_id": 30584, "store_name": "강남점", "ha_entity_id": "binary_sensor.door_01", "custom_name": "출입구 도어", "type_name": "출입문", "state_from": "off", "state_to": "on", "occurred_at": "2026-04-06T22:05:00Z" } ],
    "total_count": 1024,
    "page": 1,
    "limit": 50,
    "total_pages": 21
  }
}
```

### Edge 등록 (Edge → Backend)
| Method | Path | 설명 |
|---|---|---|
| POST | /edge/register | 최초 부팅 시 매장 등록 요청 → 외부 서버 매칭 → MQTT 계정 발급 |

### System
| Method | Path | 설명 |
|---|---|---|
| GET | /health | 헬스체크 |
| GET | /config | 시스템 설정 조회 |
| PUT | /config | 시스템 설정 수정 (external_server_url, token 등) |

---

## 공통 Response 형식

**성공:**
```json
{ "success": true, "data": {} }
```

**에러:**
```json
{ "success": false, "error": { "code": "STORE_NOT_FOUND", "message": "매장을 찾을 수 없습니다." } }
```

**HTTP 상태 코드:**
| 상태 | 의미 | 예시 error code |
|---|---|---|
| 400 | 잘못된 요청 (파라미터 누락/형식 오류) | `INVALID_REQUEST` |
| 401 | 인증 필요 (세션 없음 또는 만료) | `UNAUTHORIZED` |
| 404 | 리소스 없음 | `STORE_NOT_FOUND`, `ENTITY_NOT_FOUND` |
| 409 | 중복 (이미 존재) | `STORE_ID_CONFLICT`, `ENTITY_ALREADY_EXISTS` |
| 500 | 서버 내부 오류 | `INTERNAL_ERROR` |
| 503 | 외부 서비스 장애 (HA query timeout 등) | `HA_QUERY_TIMEOUT` |

---

## WebSocket

`ws://backend:8080/ws`

**인증**: HttpOnly 쿠키 세션 자동 포함. 세션 없으면 HTTP 401로 연결 거부.

**재연결 동작**: 재연결 시 서버가 `init` 메시지를 즉시 재전송하여 전체 상태 재동기화.
클라이언트는 재연결 후 `init` 수신 전까지 UI를 "연결 중..." 상태로 표시 권장.

### 연결 후 초기 메시지
```json
{
  "type": "init",
  "stores": [
    {
      "store_id": 30584,
      "name": "강남점",
      "importance": 4,
      "status": "online",
      "entities": [{ "ha_entity_id": "...", "custom_name": "출입구", "current_state": "off" }]
    }
  ],
  "pending_alerts": [...]
}
```

### 센서/스위치 상태 변경
```json
{
  "type": "entity.update",
  "store_id": 30584,
  "ha_entity_id": "binary_sensor.door_01",
  "custom_name": "출입구 도어",
  "type_name": "출입문",
  "state": "on",
  "timestamp": "2026-04-06T22:05:00Z"
}
```

### 알림 트리거 (영상 팝업용)
```json
{
  "type": "alert.new",
  "alert_id": 42,
  "store_id": 30584,
  "store_name": "강남점",
  "importance": 4,
  "ha_entity_id": "binary_sensor.door_01",
  "entity_name": "출입구 도어",
  "custom_name": "출입구 도어",
  "type_name": "출입문",
  "message": "출입구 도어 on",
  "state_from": "off",
  "state_to": "on",
  "stream_url": "http://1.2.3.4:1984/stream.html?src=ch1_sub",
  "is_in_schedule": true,
  "timestamp": "2026-04-06T22:05:00Z"
}
```
> `store_id`: 정수형 매장 ID (예: 30584)
> `entity_name`: `custom_name` 우선, 없으면 `ha_entity_id`
> `message`: `"{entity_name} {state_to}"` 형식의 사람이 읽기 좋은 알림 메시지
> `is_in_schedule`: `force_alert` 및 `monitoring_schedules` 기준 Backend 판단 결과.
> Frontend는 `is_in_schedule=true` 인 경우에만 소리 알림 재생. 화면 표시는 항상.

### 알림 확인 브로드캐스트 (1명 체크 → 전원 반영)
```json
{
  "type": "alert.acknowledged",
  "alert_id": 42,
  "acknowledged_by": "admin2",
  "acknowledged_at": "2026-04-06T22:05:30Z"
}
```

### Edge 상태 변경
```json
{
  "type": "store.status",
  "store_id": 30584,
  "status": "online | offline",
  "timestamp": "2026-04-06T22:00:00Z"
}
```
> `store_id`: 정수형 (INTEGER)
