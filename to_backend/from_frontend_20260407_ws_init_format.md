# WS init pending_alerts 필드명 불일치 알림

**발신**: Frontend 팀  
**수신**: Backend 팀  
**날짜**: 2026-04-07

---

## 현상

WS `init` 이벤트의 `pending_alerts`가 REST `GET /alerts` 응답 포맷으로 전달되고 있습니다.

| 필드 | REST `/alerts` 응답 | WS `alert.new` 이벤트 | WS `init.pending_alerts` (실제) |
|------|---------------------|----------------------|-------------------------------|
| 알림 ID | `id` | `alert_id` | `id` (REST 포맷) |
| 발생 시각 | `occurred_at` | `timestamp` | `occurred_at` (REST 포맷) |
| 매장명 | 없음 | `store_name` | 없음 |
| 스케줄 여부 | 없음 | `is_in_schedule` | 없음 |

## 요청

WS `init.pending_alerts` 항목을 `alert.new` 이벤트와 동일한 포맷으로 통일해주시면 감사하겠습니다:

```json
{
  "alert_id": 1,
  "store_name": "강남점",
  "timestamp": "2026-04-07T02:37:56Z",
  "is_in_schedule": true,
  ...
}
```

## 현재 대응

프론트엔드에서 `id → alert_id`, `occurred_at → timestamp` 임시 정규화 처리 중입니다.  
백엔드에서 수정 완료되면 해당 코드를 제거하겠습니다.
