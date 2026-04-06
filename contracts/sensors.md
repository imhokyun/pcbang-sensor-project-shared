# 센서 / 릴레이 계약 정의

## MQTT Topic 구조

```
pcbang/{store_id}/sensors/{sensor_id}/state        # 상태값
pcbang/{store_id}/sensors/{sensor_id}/attributes   # 메타정보 (JSON)
pcbang/{store_id}/relays/{relay_id}/state          # 릴레이 현재 상태
pcbang/{store_id}/relays/{relay_id}/set            # 릴레이 제어 명령 (backend→edge)
pcbang/{store_id}/status                           # Edge heartbeat
```

## Sensor State Payload

```json
{
  "store_id": "store_001",
  "sensor_id": "door_entrance",
  "type": "door | refrigerator_door | motion | temperature | humidity | co2",
  "state": "open | closed | on | off | <number>",
  "unit": null,
  "timestamp": "2026-04-06T12:00:00Z"
}
```

## 센서 타입 목록 (확정 필요)

| type | state 값 | unit | 설명 | 이벤트 트리거 |
|---|---|---|---|---|
| door | open / closed | - | 출입구 도어 센서 | closed→open 시 영상 팝업 |
| refrigerator_door | open / closed | - | 냉장고 문 센서 | closed→open 시 영상 팝업 |
| motion | on / off | - | 움직임 감지 | TBD |
| temperature | 숫자 | °C | 온도 | 임계값 초과 시 알림 (TBD) |
| humidity | 숫자 | % | 습도 | TBD |
| co2 | 숫자 | ppm | CO2 농도 | TBD |

## Relay Control Payload

```json
// backend → edge  (topic: pcbang/{store_id}/relays/{relay_id}/set)
{
  "command": "on | off",
  "requested_by": "user_id",
  "timestamp": "2026-04-06T12:00:00Z"
}

// edge → backend  (topic: pcbang/{store_id}/relays/{relay_id}/state)
{
  "store_id": "store_001",
  "relay_id": "relay_01",
  "state": "on | off",
  "timestamp": "2026-04-06T12:00:00Z"
}
```

## Edge Heartbeat Payload

```json
// topic: pcbang/{store_id}/status
{
  "store_id": "store_001",
  "status": "online | offline",
  "ip": "192.168.1.100",
  "go2rtc_url": "http://192.168.1.100:1984",
  "timestamp": "2026-04-06T12:00:00Z"
}
```
