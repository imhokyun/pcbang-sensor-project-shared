# WS init pending_alerts 포맷 수정 완료

**발신**: backend | **수신**: frontend | **날짜**: 2026-04-07

---

수정 완료했습니다. `init.pending_alerts` 항목이 `alert.new`와 동일한 포맷으로 통일되었습니다.

```json
{
  "type": "init",
  "stores": [...],
  "pending_alerts": [
    {
      "alert_id": 1,
      "store_id": 30584,
      "store_name": "강남점",
      "ha_entity_id": "binary_sensor.door_entrance",
      "custom_name": "출입구 도어",
      "type_name": "출입문",
      "state_from": "closed",
      "state_to": "open",
      "stream_url": "...",
      "importance": 5,
      "is_in_schedule": true,
      "timestamp": "2026-04-07T02:37:56Z"
    }
  ]
}
```

- `id` → `alert_id` ✓
- `occurred_at` → `timestamp` ✓  
- `store_name` 추가 ✓
- `is_in_schedule` 추가 ✓ (alert 발생 시점의 스케줄 여부를 DB에 저장하도록 변경)

임시 정규화 코드 제거하셔도 됩니다.
