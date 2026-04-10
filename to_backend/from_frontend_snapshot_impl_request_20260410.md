# [Backend 요청] snapshot_url 필드 구현 요청

_from: Frontend | 2026-04-10_

---

## 배경

이전 검토 회신(`from_frontend_snapshot_review_reply_20260410.md`)에서 스냅샷 방식 동의했고,
프론트엔드는 이미 아래 작업을 완료한 상태입니다.

- `Alert`, `LogEntry` 타입에 `snapshot_url?: string | null` 추가
- VideoPopup 모달: 스냅샷(왼쪽) + 실시간 영상(오른쪽) 레이아웃 구현
- 이벤트 로그 테이블: 사진보기 버튼 + 스냅샷 모달 구현
- WS `alert.new` 이벤트 → Alert 객체 매핑에 `snapshot_url` 포함

**백엔드에서 `snapshot_url` 필드를 실제로 응답에 포함시켜 주면 즉시 동작합니다.**

---

## 구현 요청 사항

### 1. WS `alert.new` 이벤트

```json
{
  "type": "alert.new",
  "data": {
    "alert_id": 123,
    "store_id": 30,
    "store_name": "강남점",
    "entity_name": "출입문 1",
    "message": "출입문 1 열림",
    "is_in_schedule": true,
    "stream_url": "http://192.168.20.30:1984/stream.html?src=ch1_sub",
    "snapshot_url": "/api/v1/snapshots/2026-04-10/30_door1_143022.jpg"
  }
}
```

- `snapshot_url` null 허용 (카메라 없는 엔티티, go2rtc offline 시)

### 2. WS `init` 이벤트 — `pending_alerts` 배열

초기 로드 시 미확인 알림 목록에도 동일하게 포함해 주세요:

```json
{
  "type": "init",
  "stores": [...],
  "pending_alerts": [
    {
      "alert_id": 123,
      ...
      "snapshot_url": "/api/v1/snapshots/2026-04-10/30_door1_143022.jpg"
    }
  ]
}
```

### 3. `GET /api/v1/logs` 응답

```json
{
  "items": [
    {
      "id": 456,
      "store_name": "강남점",
      "entity_name": "출입문 1",
      "state": "on",
      "triggered_at": "2026-04-10T14:30:22",
      "snapshot_url": "/api/v1/snapshots/2026-04-10/30_door1_143022.jpg"
    }
  ]
}
```

### 4. 이미지 서빙 엔드포인트

```
GET /api/v1/snapshots/{YYYY-MM-DD}/{filename}.jpg
```

- 인증 불필요 (StaticFiles 직접 서빙)
- 프론트에서 `${NEXT_PUBLIC_API_URL}${snapshot_url}` 방식으로 `<img src>` 사용

---

## 프론트 URL 구성

```
NEXT_PUBLIC_API_URL = https://pcbang-iot-api.multion.synology.me
snapshot_url        = /api/v1/snapshots/2026-04-10/30_door1_143022.jpg

→ 최종 URL: https://pcbang-iot-api.multion.synology.me/api/v1/snapshots/2026-04-10/30_door1_143022.jpg
```

---

## 현재 프론트 상태

| 항목 | 상태 |
|------|------|
| VideoPopup 스냅샷 레이아웃 | ✅ 완료 |
| 로그 테이블 사진보기 버튼/모달 | ✅ 완료 |
| `snapshot_url` 타입 정의 | ✅ 완료 |
| WS 이벤트 매핑 | ✅ 완료 |
| **백엔드 snapshot_url 전송** | ❌ 미구현 |
| **이미지 서빙 엔드포인트** | ❓ 미확인 |

백엔드 구현 완료되면 프론트는 추가 작업 없이 즉시 동작합니다.
