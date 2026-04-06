# API 계약 정의

Base URL: `http://backend:8080/api/v1`

---

## REST Endpoints

### Auth
| Method | Path | 설명 |
|---|---|---|
| POST | /auth/login | 로그인 (username, password → session token) |
| POST | /auth/logout | 로그아웃 |

### Stores
| Method | Path | 설명 |
|---|---|---|
| GET | /stores | 전체 매장 목록 + 온라인 상태 + 중요도 |
| GET | /stores/{store_id} | 매장 상세 |
| POST | /stores | 매장 등록 |
| PUT | /stores/{store_id} | 매장 정보 수정 (name, address, device_sn, importance, force_alert) |
| DELETE | /stores/{store_id} | 매장 삭제 |

### Cameras
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/cameras | 카메라 채널 목록 |
| POST | /stores/{store_id}/cameras | 카메라 추가 → go2rtc API 즉시 반영 |
| PUT | /stores/{store_id}/cameras/{channel} | RTSP URL 수정 → go2rtc API 즉시 반영 |
| DELETE | /stores/{store_id}/cameras/{channel} | 카메라 삭제 → go2rtc stream 제거 |

> go2rtc 연동: `PUT http://{go2rtc_url}/api/streams/{stream_name}` (Backend 내부 호출)

### HA Entities
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/ha/entities | HA에서 전체 entity 목록 조회 (실시간 query) |
| GET | /stores/{store_id}/entities | 등록된 entity 목록 + 현재 상태 |
| POST | /stores/{store_id}/entities | entity 등록 (ha_entity_id, custom_name, type_id, triggers_alert) |
| PUT | /stores/{store_id}/entities/{ha_entity_id} | entity 수정 |
| DELETE | /stores/{store_id}/entities/{ha_entity_id} | entity 제거 |

### Entity Types
| Method | Path | 설명 |
|---|---|---|
| GET | /entity-types | 전체 타입 목록 (기본 + 사용자 추가) |
| POST | /entity-types | 사용자 정의 타입 추가 |
| DELETE | /entity-types/{id} | 사용자 정의 타입 삭제 (기본값 삭제 불가) |

### Relays (switch entity 제어)
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/relays | 스위치 entity 목록 + 현재 상태 |
| POST | /stores/{store_id}/relays/{ha_entity_id}/command | on/off 제어 → MQTT publish |

### Monitoring Schedules
| Method | Path | 설명 |
|---|---|---|
| GET | /stores/{store_id}/schedules | 요일별 관제 시간 목록 |
| PUT | /stores/{store_id}/schedules/{day_of_week} | 수동 시간 수정 (is_manual=true) |
| POST | /stores/{store_id}/schedules/sync | 외부 서버에서 즉시 폴링하여 업데이트 |

### Alerts
| Method | Path | 설명 |
|---|---|---|
| GET | /alerts | 미확인 알림 목록 (전체 매장) |
| POST | /alerts/{alert_id}/acknowledge | 알림 확인 처리 → WS 브로드캐스트 |

### Logs
| Method | Path | 설명 |
|---|---|---|
| GET | /logs | 이벤트 이력 조회 |

**Query params**: `store_id`, `type_name`, `state_from`, `state_to`, `from` (datetime), `to` (datetime), `page`, `limit`

### Edge 등록 (Edge → Backend)
| Method | Path | 설명 |
|---|---|---|
| POST | /edge/register | 최초 부팅 시 매장 등록 요청 → 외부 서버 매칭 → MQTT 계정 발급 |

### System
| Method | Path | 설명 |
|---|---|---|
| GET | /health | 헬스체크 |
| GET | /config | 시스템 설정 조회 |
| PUT | /config | 시스템 설정 수정 (external_server_url, token 등) |

---

## 공통 Response 형식

```json
{
  "success": true,
  "data": {},
  "error": null
}
```

---

## WebSocket

`ws://backend:8080/ws`

### 연결 후 초기 메시지
```json
{
  "type": "init",
  "stores": [
    {
      "store_id": "store_001",
      "name": "강남점",
      "importance": 4,
      "status": "online",
      "entities": [{ "ha_entity_id": "...", "custom_name": "출입구", "current_state": "closed" }]
    }
  ],
  "pending_alerts": [...]
}
```

### 센서/스위치 상태 변경
```json
{
  "type": "entity.update",
  "store_id": "store_001",
  "ha_entity_id": "binary_sensor.door_01",
  "custom_name": "출입구 도어",
  "type_name": "출입문",
  "state": "open",
  "timestamp": "2026-04-06T22:05:00Z"
}
```

### 알림 트리거 (영상 팝업용)
```json
{
  "type": "alert.new",
  "alert_id": 42,
  "store_id": "store_001",
  "store_name": "강남점",
  "importance": 4,
  "ha_entity_id": "binary_sensor.door_01",
  "custom_name": "출입구 도어",
  "type_name": "출입문",
  "state_from": "closed",
  "state_to": "open",
  "stream_url": "http://1.2.3.4:1984/stream/store_001_ch01",
  "timestamp": "2026-04-06T22:05:00Z"
}
```

### 알림 확인 브로드캐스트 (1명 체크 → 전원 반영)
```json
{
  "type": "alert.acknowledged",
  "alert_id": 42,
  "acknowledged_by": "admin2",
  "acknowledged_at": "2026-04-06T22:05:30Z"
}
```

### Edge 상태 변경
```json
{
  "type": "store.status",
  "store_id": "store_001",
  "status": "online | offline",
  "timestamp": "2026-04-06T22:00:00Z"
}
```
