# [Backend → Edge] 릴레이 제어 명령 미반응 확인 요청

**발신**: Backend  
**수신**: Edge  
**일시**: 2026-04-07

---

## 현상

Frontend에서 정문 잠금장치 on/off 제어 시 Edge가 반응하지 않음.

## Backend 확인 결과

아래 3단계 모두 정상 확인됨:

| 단계 | 결과 |
|------|------|
| Frontend → Backend API (`POST /relays/.../command`) | ✅ 200 OK |
| Backend → MQTT publish | ✅ 브로커 도달 확인 |
| Edge MQTT 접속 상태 (client: `30584`) | ✅ 연결 중 |

MQTT 브로커에 실제로 도달한 메시지:
```json
topic: pcbang/30584/relays/switch.jeongmun_jamgeumjangci/set

{
  "command": "on",
  "requested_by": "admin1",
  "timestamp": "2026-04-07T05:42:20.360363+00:00"
}
```

---

## 확인 요청

1. **토픽 구독 여부** — `pcbang/30584/relays/switch.jeongmun_jamgeumjangci/set` 을 구독하고 있는지 확인
2. **HA entity 존재 여부** — `switch.jeongmun_jamgeumjangci` 가 HA에 실제로 등록되어 있는지 확인
3. **command 처리 로직** — `"command": "on"` / `"command": "off"` 수신 시 `hass.services.call("switch", "turn_on/turn_off", ...)` 호출하는지 확인

계약 문서(`contracts/mqtt.md`) 기준:
```
topic   : pcbang/{store_id}/relays/{ha_entity_id}/set
payload : { "command": "on | off", "requested_by": "...", "timestamp": "..." }
```
