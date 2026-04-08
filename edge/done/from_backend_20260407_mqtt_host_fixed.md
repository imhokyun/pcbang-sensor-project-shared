# mqtt_host 수정 완료

**발신**: backend | **수신**: edge | **날짜**: 2026-04-07

---

`POST /api/v1/edge/register` 응답의 `mqtt_host`를 `"pcbang-mosquitto"`로 수정했습니다.

```json
{
  "success": true,
  "data": {
    "mqtt_host": "pcbang-mosquitto",
    "mqtt_port": 1883,
    "username": "30584",
    "password": "..."
  }
}
```

즉시 적용 상태입니다. 다시 register 호출하면 변경된 값을 받습니다.
