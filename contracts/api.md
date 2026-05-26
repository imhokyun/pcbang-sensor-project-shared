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
| GET | /alerts | 미확인 알림 목록 (전체 매장, `type_id` 필터 지원) |
| POST | /alerts/{alert_id}/acknowledge | 단일 알림 확인 처리 → WS `alert.acknowledged` |
| POST | /alerts/acknowledge-all | 일괄 확인 처리 → WS `alert.acknowledged_batch` |

**GET /alerts query params**:
- `type_id` (선택): `entity_types.id`. 지정 시 해당 카테고리만 반환. 미지정 시 전체 미확인 알림.

**POST /alerts/acknowledge-all request body** (셋 중 최소 하나 필수 — 빈 body는 400):
```json
{ "ids": [1, 2, 3] }                 // 명시적 ID 배열
{ "store_id": 30584 }                // 해당 매장의 미확인 전체
{ "type_id": 3 }                     // 해당 카테고리의 미확인 전체
{ "store_id": 30584, "type_id": 3 }  // AND 필터
```

**response**:
```json
{ "success": true, "data": { "acknowledged_count": 12, "acknowledged_ids": [1, 2, ..., 12] } }
```
> 이미 확인된 알림은 `acknowledged_by IS NULL` 가드로 자동 제외 — 동시 호출 안전.
> `400 INVALID_BODY`: body가 비었거나 모든 필터가 비었을 때.

### Watch Alerts (관심로그)
| Method | Path | 설명 |
|---|---|---|
| POST | /watch-alerts | 알림을 관심로그로 등록 (자동 ack 트랜잭션) → WS `alert.acknowledged` |
| GET | /watch-alerts | 관심로그 목록 (status 필터, alert 스냅샷 join) |
| PATCH | /watch-alerts/{id} | 상태/메모 변경 (status 전이 시 resolved_* 자동 채움/초기화) |
| DELETE | /watch-alerts/{id} | hard delete (원본 alert_events 보존) |

**POST /watch-alerts request body**:
```json
{ "alert_id": 4521, "note": "냉장고 3분 이상 열림, CCTV 다시 보기 필요" }
```
- `note`는 필수 (빈 문자열 400). 길이 최대 500자.
- 트랜잭션: watch insert + alert ack (alert가 이미 ack된 경우 watch만 insert).
- 응답 후 기존 `alert.acknowledged` WS broadcast 발생 → 다른 관제자 메인 화면에서 즉시 사라짐.
- `404 ALERT_NOT_FOUND`: alert_id 미존재.

**GET /watch-alerts query params**:
- `status` (선택): `pending` | `resolved` | `all`. 기본 `all`. 그 외 값은 400.

**PATCH /watch-alerts/{id} request body** (둘 다 선택, 최소 하나 필수):
```json
{ "status": "resolved" }
{ "note": "현장 출동 후 정상 확인" }
{ "status": "resolved", "note": "..." }
```
- `pending → resolved`: `resolved_at = NOW()`, `resolved_by = 현재 사용자` 자동 채움.
- `resolved → pending`: `resolved_at = NULL`, `resolved_by = NULL` 자동 초기화.
- status 값 검증: `pending` / `resolved` 외 400.

**공통 응답 (POST/GET/PATCH)**:
```json
{
  "success": true,
  "data": {
    "watch_id": 12,
    "status": "pending",
    "note": "냉장고 3분 이상 열림",
    "created_by": { "id": 1, "username": "admin1" },
    "created_at": "2026-05-26T22:14:33Z",
    "resolved_by": null,
    "resolved_at": null,
    "alert": {
      "alert_id": 4521,
      "store_id": 30263,
      "type_id": 5,
      "type_name": "냉장고 도어",
      "custom_name": "주류 냉장고 #2",
      "snapshot_url": "https://.../snap.jpg",
      "stream_url": "https://.../stream",
      "occurred_at": "2026-05-26T22:13:50Z",
      "importance": 4
    }
  }
}
```
GET은 `data`가 위 object의 배열, `created_at DESC` 정렬.

**DELETE /watch-alerts/{id} 응답**:
```json
{ "success": true }
```
원본 `alert_events`는 보존 (감사 추적).

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

### CTRL 동기화 (외부 webhook 수신 + 수동 동기화)

| Method | Path | 설명 |
|---|---|---|
| POST | /webhooks/ctrl/store | n8n 중계로 CTRL 매장 변경 webhook 수신 (`X-WEBHOOK-SECRET` 인증) |
| POST | /stores/{store_id}/schedules/sync | 운영자 수동 동기화 — CTRL `/api/store/store_detail` 호출 후 monitoring_schedules 반영 |

#### POST /webhooks/ctrl/store

**인증**: `X-WEBHOOK-SECRET` 헤더 + 우리 `.env`의 `CTRL_WEBHOOK_SECRET` 검증. 불일치/누락 시 `401`. 세션 쿠키 불필요 (서버-서버 호출).

**요청 본문** (CTRL 원본 passthrough, n8n이 변형 없이 전달):
```json
{
  "storeNo": 30584,
  "storeNm": "강남점",
  "eventType": "updated",
  "controlTimes": [
    { "daySn": 2, "dayNm": "월", "beginTime": "03:00", "endTime": "09:00" },
    { "daySn": 3, "dayNm": "화", "beginTime": "03:00", "endTime": "10:00" },
    { "daySn": 4, "dayNm": "수", "beginTime": "03:00", "endTime": "09:00" },
    { "daySn": 5, "dayNm": "목", "beginTime": "03:00", "endTime": "10:00" },
    { "daySn": 6, "dayNm": "금", "beginTime": "03:00", "endTime": "10:00" },
    { "daySn": 7, "dayNm": "토", "beginTime": "05:00", "endTime": "11:00" },
    { "daySn": 1, "dayNm": "일", "beginTime": "",      "endTime": ""      }
  ],
  "delAt": "N",
  "timestamp": "2026-05-26T12:37:17.874"
}
```

> 본문에는 매장명·주소·담당자·연락처·장비 IP/Pwd 필드도 포함되지만 우리는 **무시**한다 (CTRL이 source-of-truth).

**필수 필드**: `storeNo`, `eventType`, `controlTimes`. 누락 시 `422`.

**`eventType` 분기**:
| 값 | 우리 동작 |
|---|---|
| `created` | `controlTimes` → `monitoring_schedules` 반영. `stores` row 미존재 시 무시 + 로그 (Edge 등록 우선). |
| `updated` | controlTimes만 추출해 반영. |
| `time_updated` | 동일하게 controlTimes 반영. |
| `deleted` | 로그만 남기고 DB 변경 X (운영 실수 보호). |

**`controlTimes[]` 요일 매핑**: CTRL `daySn` (1=일, 2=월, ..., 7=토) ↔ 우리 `monitoring_schedules.day_of_week` (0=월, ..., 6=일, Python 표준). 변환 `(daySn + 5) % 7`.

**응답**:
```json
{
  "success": true,
  "data": {
    "store_id": 30584,
    "event_type": "updated",
    "applied": "schedules",
    "updated_count": 7,
    "skipped_manual": 0
  }
}
```

- `updated_count`: 실제 갱신된 `monitoring_schedules` row 수 (`is_manual=0`만)
- `skipped_manual`: 운영자가 수동 설정한(`is_manual=1`) 행으로 보존된 수
- 매장 미존재/`deleted` 이벤트 시: `applied`는 `"none"`, `updated_count`/`skipped_manual` = 0

#### POST /stores/{store_id}/schedules/sync (변경 — 위임)

**인증**: 기존 세션 쿠키 그대로.

**동작**: 내부적으로 `services/ctrl_sync.sync_store_schedules(store_id, db)` 호출 → CTRL `/api/store/store_detail?storeNo=N` 패치 → controlTimes 반영. `services/external_poll.py` 는 **DEPRECATED** 처리, 더는 호출되지 않는다.

**응답** (기존 클라이언트 호환):
```json
{ "success": true, "data": { "updated_count": 7, "skipped_manual": 0 } }
```

**에러**:
| HTTP | code | 발생 조건 |
|---|---|---|
| 502 | `CTRL_AUTH_FAILED` | CTRL `X-SERVER-API-KEY` 무효 또는 401 |
| 504 | `CTRL_TIMEOUT` | CTRL 응답 10초 초과 |
| 404 | `STORE_NOT_FOUND` | 우리 DB에 `stores.store_id` 미존재 |
| 502 | `CTRL_STORE_NOT_FOUND` | CTRL이 해당 매장 `400 item not exist` 반환 |

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
  "type_id": 1,
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
> `type_id`: `entity_types.id` — Frontend 카테고리 필터의 식별자. `type_id IS NULL` 알림은 (`entity.type_id` 미설정) 발생하지 않음 (alert_triggers 매칭 불가).
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

### 알림 일괄 확인 브로드캐스트 (일괄확인 → 전원 반영)
```json
{
  "type": "alert.acknowledged_batch",
  "alert_ids": [1, 2, 3, 12, 18],
  "acknowledged_by": "admin2",
  "acknowledged_at": "2026-04-06T22:05:30Z"
}
```
> 단일 메시지에 ack된 ids 배열. Frontend는 한 번에 모두 리스트에서 제거.
> 이미 확인된 알림은 서버측 IS NULL 가드로 제외되므로 `alert_ids`는 실제로 이번 호출에서 처리된 것만.

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
