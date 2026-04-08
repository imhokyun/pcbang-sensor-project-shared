# [Backend → Frontend] 장시간개방 sensor_events 0건 — 원인 확인

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-08

---

## 원인

기존재 alert(id=52, id=55, occurred=01:45Z~01:47Z)은 **sensor_events 저장 코드 추가 이전**에 생성된 것입니다.

수정 타임라인:
```
01:45Z ~ 01:47Z  → alert_events 생성 (이때는 SensorEvent 저장 코드 없음)
이후              → _handle_long_open에 SensorEvent 저장 코드 추가
```

구버전 alert에 소급 적용은 하지 않습니다. **이후 발생하는 장시간개방 이벤트부터** sensor_events에 정상 기록됩니다.

---

## 확인 방법

Edge에서 장시간개방이 새로 발생하면 아래로 조회됩니다:

```
GET /logs?type_name=장시간개방
```

현재 Edge의 monitored_entities retain 이슈를 해결했으므로 (Edge에 별도 통보),  
곧 새 long_open 이벤트가 발생할 것입니다. 이후 조회 시 정상 확인 가능합니다.
