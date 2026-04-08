# [Frontend → Backend] 장시간개방 sensor_events 미저장 이슈

**발신**: Frontend  
**수신**: Backend  
**일시**: 2026-04-08

---

## 현상

`alert_events`에는 장시간개방 알림이 정상 생성되어 있으나,  
`GET /logs` (`sensor_events`) 에서는 `type_name = "장시간개방"` 항목이 **0건**입니다.

```
GET /logs?type_name=장시간개방
→ { "total_count": 0, "items": [] }
```

`alert_events` 확인 결과:
- id=52: store=30584, entity=binary_sensor.naengjanggo_1, occurred=2026-04-08T01:45Z
- id=55: store=30584, entity=binary_sensor.naengjanggo_1, occurred=2026-04-08T01:47Z

→ 알림은 2건 생성됐지만 이벤트 로그에는 없음

---

## 요청

`_handle_long_open` 처리 시 `sensor_events`에 `type_name="장시간개방"` 행이 저장되지 않는 원인 확인 및 수정 부탁드립니다.

Frontend는 표시 처리가 완료되어 있으므로, 데이터만 정상 저장되면 바로 노출됩니다.

---

## 참고

- Backend 문서 `from_backend_20260408_long_open_log_display.md` 기반으로 Frontend 표시 처리 완료
- `type_name === "장시간개방"` 일 때: 노란색(warning) 배지 + "장시간 개방" 텍스트 표시 구현됨
