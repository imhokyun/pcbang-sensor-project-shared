# API 계약 정의

Base URL: `http://backend:8080/api/v1`

## REST Endpoints

### Stores
| Method | Path | 설명 |
|---|---|---|
| GET | /stores | 전체 매장 목록 + 현재 온라인 상태 |
| GET | /stores/{store_id} | 매장 상세 (센서 목록, go2rtc URL 포함) |
| POST | /stores | 매장 등록 |
| PUT | /stores/{store_id} | 매장 정보 수정 |

### Sensors
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/sensors | 매장 센서 목록 + 최신 상태 |
| GET | /stores/{store_id}/sensors/{sensor_id}/history | 센서 이벤트 이력 (페이징) |

### Relays
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/relays | 릴레이 목록 + 현재 상태 |
| POST | /stores/{store_id}/relays/{relay_id}/command | 릴레이 on/off 명령 |

### Events
| Method | Path | 설명 |
|---|---|---|
| GET | /events | 전체 매장 이벤트 이력 (필터: store_id, type, from, to) |

### System
| Method | Path | 설명 |
|---|---|---|
| GET | /health | 헬스체크 |

## 공통 Response 형식

```json
{
  "success": true,
  "data": {},
  "error": null
}
```

## WebSocket

`ws://backend:8080/ws`

### 연결 시 초기 메시지
```json
{ "type": "init", "stores": [...] }
```

### 실시간 이벤트 메시지
```json
{
  "type": "sensor.update | relay.update | store.status",
  "store_id": "store_001",
  "sensor_id": "door_entrance",
  "state": "open",
  "timestamp": "2026-04-06T12:00:00Z"
}
```

### 트리거 이벤트 (영상 팝업용)
```json
{
  "type": "event.trigger",
  "store_id": "store_001",
  "sensor_type": "door | refrigerator_door",
  "state_from": "closed",
  "state_to": "open",
  "stream_url": "http://192.168.1.100:1984/stream/cam1",
  "timestamp": "2026-04-06T12:00:00Z"
}
```
