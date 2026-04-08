# Backend API 준비 완료 — 연동 가이드

**발신**: backend | **수신**: frontend | **날짜**: 2026-04-07

---

## 접근 주소

| 항목 | 주소 |
|---|---|
| REST API Base | `http://localhost:8080/api/v1` |
| WebSocket | `ws://localhost:8080/ws` |
| API 문서 (Swagger) | `http://localhost:8080/docs` |
| 헬스체크 | `http://localhost:8080/api/v1/health` |

> Docker 컨테이너 환경에서는 `localhost` → `pcbang-backend` (서비스명)

---

## 인증

모든 API는 **HttpOnly 쿠키 세션** 방식. 아래 순서로 진행:

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "username": "admin1", "password": "admin1234" }
```

성공 시 `Set-Cookie: session_id=...; HttpOnly; SameSite=Lax` 자동 설정.  
이후 모든 요청에 쿠키가 자동 포함되므로 별도 처리 불필요.

**테스트 계정**: `admin1` ~ `admin5` / 비밀번호: `admin1234`

---

## WebSocket

```js
const ws = new WebSocket("ws://localhost:8080/ws");
// 로그인 후 쿠키가 자동 포함됨

ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  switch (msg.type) {
    case "init":            // 최초 연결 시: 전체 매장 상태 + 미확인 알림
    case "entity.update":   // 센서/스위치 상태 변경
    case "alert.new":       // 새 알림 (is_in_schedule=true 이면 소리 알림)
    case "alert.acknowledged": // 알림 확인 (다른 관제원이 확인해도 전파)
    case "store.status":    // 매장 online/offline 전환
  }
};
```

세션 없이 연결 시 **코드 1008**로 즉시 종료.  
재연결 시 서버가 `init`을 즉시 재전송하므로 상태 재동기화 자동 처리.

---

## 주요 API 요약

### 매장
```
GET    /api/v1/stores                           매장 목록 (importance 내림차순)
GET    /api/v1/stores/{store_id}                매장 상세
POST   /api/v1/stores                           매장 등록
PUT    /api/v1/stores/{store_id}                매장 수정 (force_alert: null|0|1 포함)
DELETE /api/v1/stores/{store_id}                매장 삭제
```

### 카메라
```
GET    /api/v1/stores/{store_id}/cameras
POST   /api/v1/stores/{store_id}/cameras        go2rtc 자동 등록
PUT    /api/v1/stores/{store_id}/cameras/{ch}   go2rtc 즉시 반영
DELETE /api/v1/stores/{store_id}/cameras/{ch}
```

### 센서/릴레이
```
GET    /api/v1/stores/{store_id}/ha/entities    HA에서 실시간 조회 (최대 10초)
GET    /api/v1/stores/{store_id}/entities       등록된 entity 목록
POST   /api/v1/stores/{store_id}/entities
PUT    /api/v1/stores/{store_id}/entities/{ha_entity_id}
DELETE /api/v1/stores/{store_id}/entities/{ha_entity_id}

GET    /api/v1/stores/{store_id}/relays
POST   /api/v1/stores/{store_id}/relays/{ha_entity_id}/command
       body: { "command": "on" | "off" }
```

### 관제 스케줄
```
GET    /api/v1/stores/{store_id}/schedules
PUT    /api/v1/stores/{store_id}/schedules/{day_of_week}
       body: { "start_time": "22:00", "end_time": "12:00", "is_active": 1 }
       day_of_week: 0=월 ~ 6=일 / end < start 이면 익일까지
POST   /api/v1/stores/{store_id}/schedules/sync  외부서버 즉시 폴링
```

### 알림
```
GET    /api/v1/alerts                           미확인 알림 전체
POST   /api/v1/alerts/{alert_id}/acknowledge    확인 처리 → WS 브로드캐스트
```

### 로그
```
GET    /api/v1/logs?store_id=&type_name=&from=&to=&page=1&limit=50
```

### 기타
```
GET    /api/v1/entity-types
POST   /api/v1/entity-types
DELETE /api/v1/entity-types/{id}    기본 타입(출입문/냉장고/카운터/기타) 삭제 불가
GET    /api/v1/config
PUT    /api/v1/config
```

---

## 공통 응답 형식

**성공**
```json
{ "success": true, "data": { ... } }
```

**에러**
```json
{ "success": false, "error": { "code": "STORE_NOT_FOUND", "message": "..." } }
```

주요 HTTP 상태 코드: `401` 인증 없음, `404` 리소스 없음, `409` 중복, `503` Edge 오프라인

---

## 주요 비즈니스 로직

- **`alert.new` → `is_in_schedule`**: `true`이면 소리 알림 재생, `false`이면 화면 표시만
- **중요도 4 이상 매장**: 알림 row 붉은 배경 처리 권장
- **매장 offline**: heartbeat 90초 미수신 → `store.status` WS 이벤트 발생
- **세션 TTL**: 24시간 rolling (요청마다 자동 연장)
