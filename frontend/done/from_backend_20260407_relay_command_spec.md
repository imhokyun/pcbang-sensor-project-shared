# [Backend → Frontend] 릴레이 제어 API 스펙 안내

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

---

## 에러 원인

`POST /api/v1/stores/30584/relays/switch.jeongmun_jamgeumjangci/command` 요청이 **422 Unprocessable Entity**로 실패하고 있습니다.

422는 request body 구조가 잘못됐을 때 발생합니다. `command` 필드가 없거나 필드명이 다른 경우입니다.

---

## 올바른 요청 형식

```
POST /api/v1/stores/{store_id}/relays/{ha_entity_id}/command
Content-Type: application/json
```

**Body:**
```json
{ "command": "on" }
```
또는
```json
{ "command": "off" }
```

- 필드명: 반드시 `"command"` (다른 이름 불가)
- 허용 값: `"on"` 또는 `"off"` 만 유효 (그 외 값은 400 에러)
- 잠금장치 열기 → `"on"`, 잠그기 → `"off"` 로 사용하세요

---

## 예시

```javascript
// 정문 잠금장치 열기
fetch('/api/v1/stores/30584/relays/switch.jeongmun_jamgeumjangci/command', {
  method: 'POST',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ command: 'on' })
})
```

---

## 릴레이 목록 조회

등록된 switch 목록은 아래 API로 조회합니다:
```
GET /api/v1/stores/{store_id}/relays
```

현재 강남점(30584) 등록된 릴레이:
- `switch.jeongmun_jamgeumjangci` — 정문 잠금장치
