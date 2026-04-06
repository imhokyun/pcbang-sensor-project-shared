# DB 스키마 설계

## 엔진
- SQLite (WAL 모드)
- 단일 파일, Backend 전용 writer
- Mosquitto는 `mqtt_credentials` 테이블만 read-only 접근 (`mosquitto-go-auth` 플러그인)

---

## 테이블 정의

### stores
매장 기본 정보 및 Edge 연결 상태

```sql
CREATE TABLE stores (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id      TEXT UNIQUE NOT NULL,   -- "store_001"
  name          TEXT NOT NULL,          -- "강남점"
  address       TEXT,
  go2rtc_url    TEXT,                   -- "http://1.2.3.4:1984"
  status        TEXT DEFAULT 'offline', -- online / offline
  last_seen_at  DATETIME,
  registered_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### sensors
매장별 센서 목록 및 현재 상태

```sql
CREATE TABLE sensors (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id        TEXT NOT NULL REFERENCES stores(store_id),
  sensor_id       TEXT NOT NULL,   -- "door_entrance"
  type            TEXT NOT NULL,   -- door / refrigerator_door / motion / temperature / humidity / co2 / ...
  name            TEXT,            -- "출입구 도어"
  current_state   TEXT,            -- open / closed / on / off / 숫자
  last_updated_at DATETIME,
  UNIQUE(store_id, sensor_id)
);
```

### relays
매장별 릴레이 목록 및 현재 상태

```sql
CREATE TABLE relays (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id        TEXT NOT NULL REFERENCES stores(store_id),
  relay_id        TEXT NOT NULL,
  name            TEXT,
  current_state   TEXT DEFAULT 'off',  -- on / off
  last_updated_at DATETIME,
  UNIQUE(store_id, relay_id)
);
```

### sensor_events
전체 센서 상태 변화 이력 (3개월 보관)

```sql
CREATE TABLE sensor_events (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id    TEXT NOT NULL,
  sensor_id   TEXT NOT NULL,
  type        TEXT NOT NULL,
  state_from  TEXT,
  state_to    TEXT NOT NULL,
  occurred_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_sensor_events_store ON sensor_events(store_id, occurred_at);
CREATE INDEX idx_sensor_events_time ON sensor_events(occurred_at);
```

### alert_events
트리거된 알림 (3개월 보관, 매일 14:00 자동 삭제)

```sql
CREATE TABLE alert_events (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id         TEXT NOT NULL,
  sensor_id        TEXT NOT NULL,
  sensor_type      TEXT NOT NULL,
  state_from       TEXT,
  state_to         TEXT NOT NULL,
  stream_url       TEXT,             -- 발생 시점 go2rtc URL 스냅샷
  acknowledged_by  INTEGER REFERENCES users(id),
  acknowledged_at  DATETIME,
  occurred_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_alert_events_store ON alert_events(store_id, occurred_at);
```

### alert_triggers
어떤 센서 타입 / 상태 전환이 알림을 발생시키는지 정의 (추후 확장)

```sql
CREATE TABLE alert_triggers (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  sensor_type  TEXT NOT NULL,   -- door / refrigerator_door / motion / ...
  state_from   TEXT,            -- NULL = any
  state_to     TEXT NOT NULL,   -- open / on / ...
  is_active    INTEGER DEFAULT 1
);
-- 초기값 예시:
-- INSERT INTO alert_triggers(sensor_type, state_from, state_to) VALUES ('door', 'closed', 'open');
-- INSERT INTO alert_triggers(sensor_type, state_from, state_to) VALUES ('refrigerator_door', 'closed', 'open');
```

### mqtt_credentials
매장별 MQTT 인증 정보 (mosquitto-go-auth가 read)

```sql
CREATE TABLE mqtt_credentials (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id      TEXT UNIQUE NOT NULL REFERENCES stores(store_id),
  username      TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,   -- bcrypt
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### monitoring_schedules
매장별 요일별 관제 시간 (외부 서버 폴링 or 수동 설정)

```sql
CREATE TABLE monitoring_schedules (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  store_id    TEXT NOT NULL REFERENCES stores(store_id),
  day_of_week INTEGER NOT NULL,  -- 0=월 1=화 2=수 3=목 4=금 5=토 6=일
  start_time  TEXT NOT NULL,     -- "HH:MM" (22:00)
  end_time    TEXT NOT NULL,     -- "HH:MM" (12:00) — end < start 이면 익일까지
  is_active   INTEGER DEFAULT 1, -- 해당 요일 관제 여부
  is_manual   INTEGER DEFAULT 0, -- 0=폴링값, 1=수동설정
  updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(store_id, day_of_week)
);
```

> **자정 넘김 처리**: `end_time < start_time` 이면 익일 end_time까지로 판단.
> 예: start=22:00 / end=12:00 → 22:00~익일 12:00

### users
관제 대시보드 계정

```sql
CREATE TABLE users (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  username      TEXT UNIQUE NOT NULL,  -- admin1 ~ admin5
  password_hash TEXT NOT NULL,
  display_name  TEXT
);
```

### system_config
시스템 설정 key-value 저장소

```sql
CREATE TABLE system_config (
  key        TEXT PRIMARY KEY,
  value      TEXT,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
-- 초기 키 목록:
-- external_server_url    외부 관제서버 API URL (추후 입력)
-- external_server_token  인증 토큰 (추후 입력)
-- polling_interval_min   관제시간 폴링 주기 (분)
-- alert_cleanup_hour     알림 삭제 시간 (기본: 14)
-- alert_retention_days   알림 보관 기간 (기본: 90)
```

---

## 자동 삭제 스케줄

매일 14:00 (APScheduler):
```python
DELETE FROM sensor_events WHERE occurred_at < datetime('now', '-90 days');
DELETE FROM alert_events  WHERE occurred_at < datetime('now', '-90 days');
```

---

## 미결: alert_triggers 전체 목록
Edge 기기 구성 확정 후 채워야 함. → `docs/tbd.md` 참조
