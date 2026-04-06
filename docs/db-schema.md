# DB 스키마 설계

## 엔진
- **PostgreSQL** (Docker), Backend 전용 writer
- Mosquitto는 `mqtt_credentials` 테이블만 read-only (`mosquitto-go-auth` 플러그인, PostgreSQL 백엔드 사용)

---

## 테이블 정의

### stores
```sql
CREATE TABLE stores (
  id              SERIAL PRIMARY KEY,
  store_id        TEXT UNIQUE NOT NULL,   -- "store_001"
  name            TEXT NOT NULL,          -- "강남점"
  address         TEXT,
  device_sn       TEXT,                   -- Edge 기기 시리얼 번호
  go2rtc_url      TEXT,                   -- "http://1.2.3.4:1984"
  importance      INTEGER DEFAULT 3,      -- 1(낮음)~5(높음), 4이상=메인화면 붉은 배경
  force_alert     INTEGER DEFAULT NULL,   -- NULL=스케줄따름, 0=강제OFF, 1=강제ON
  status          TEXT DEFAULT 'offline', -- online / offline
  last_seen_at    TIMESTAMPTZ,
  registered_at   TIMESTAMPTZ DEFAULT NOW()
);
```

### cameras
매장별 DVR 카메라 채널 (6~14개)
```sql
CREATE TABLE cameras (
  id          SERIAL PRIMARY KEY,
  store_id    TEXT NOT NULL REFERENCES stores(store_id),
  channel     INTEGER NOT NULL,     -- 1~14
  name        TEXT,                 -- "입구", "카운터" (사용자 지정)
  rtsp_main   TEXT,                 -- rtsp://ip:554/channel/01/main
  rtsp_sub    TEXT,                 -- rtsp://ip:554/channel/01/sub
  stream_name TEXT,                 -- go2rtc stream key: "store_001_ch01"
  is_active   INTEGER DEFAULT 1,
  UNIQUE(store_id, channel)
);
```
> **go2rtc 연동**: 카메라 저장/수정 시 Backend가 `PUT http://{go2rtc_url}/api/streams/{stream_name}` 호출하여 실시간 반영. config.yaml reload 불필요.

### entity_types
사용자 정의 센서/릴레이 타입 (기본값 + 사용자 추가 가능)
```sql
CREATE TABLE entity_types (
  id         SERIAL PRIMARY KEY,
  name       TEXT UNIQUE NOT NULL,  -- "냉장고", "출입문", "카운터", "기타"
  is_default INTEGER DEFAULT 0      -- 1=기본 제공, 0=사용자 추가
);
-- 초기값:
-- INSERT INTO entity_types(name, is_default) VALUES ('출입문', 1);
-- INSERT INTO entity_types(name, is_default) VALUES ('냉장고', 1);
-- INSERT INTO entity_types(name, is_default) VALUES ('카운터', 1);
-- INSERT INTO entity_types(name, is_default) VALUES ('기타', 1);
```

### store_entities
매장에서 선택한 HA entity (센서/스위치) 목록
```sql
CREATE TABLE store_entities (
  id             SERIAL PRIMARY KEY,
  store_id       TEXT NOT NULL REFERENCES stores(store_id),
  ha_entity_id   TEXT NOT NULL,       -- HA entity_id: "binary_sensor.door_01"
  entity_kind    TEXT NOT NULL,       -- sensor / switch
  custom_name    TEXT,                -- "출입구 도어" (사용자 지정 명칭)
  type_id        INTEGER REFERENCES entity_types(id),
  triggers_alert INTEGER DEFAULT 0,   -- 1=상태변화 시 alert 발생 대상
  current_state  TEXT,
  last_updated_at TIMESTAMPTZ,
  is_active      INTEGER DEFAULT 1,
  UNIQUE(store_id, ha_entity_id)
);
```

### sensor_events
전체 entity 상태 변화 이력 (3개월 보관)
```sql
CREATE TABLE sensor_events (
  id           SERIAL PRIMARY KEY,
  store_id     TEXT NOT NULL,
  ha_entity_id TEXT NOT NULL,
  entity_kind  TEXT NOT NULL,      -- sensor / switch
  type_name    TEXT,               -- entity_types.name 스냅샷
  custom_name  TEXT,               -- 변경될 수 있으므로 이벤트 시점 값 저장
  state_from   TEXT,
  state_to     TEXT NOT NULL,
  occurred_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_sensor_events_store ON sensor_events(store_id, occurred_at);
CREATE INDEX idx_sensor_events_time  ON sensor_events(occurred_at);
```

### alert_events
알림 트리거된 이벤트 (3개월 보관)
```sql
CREATE TABLE alert_events (
  id              SERIAL PRIMARY KEY,
  store_id        TEXT NOT NULL,
  ha_entity_id    TEXT NOT NULL,
  type_name       TEXT,            -- 이벤트 시점 타입명 스냅샷
  custom_name     TEXT,            -- 이벤트 시점 명칭 스냅샷
  state_from      TEXT,
  state_to        TEXT NOT NULL,
  stream_url      TEXT,            -- 발생 시점 go2rtc stream URL 스냅샷
  importance      INTEGER,         -- 발생 시점 매장 중요도 스냅샷
  acknowledged_by INTEGER REFERENCES users(id),
  acknowledged_at TIMESTAMPTZ,
  occurred_at     TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_alert_events_store ON alert_events(store_id, occurred_at);
CREATE INDEX idx_alert_events_ack   ON alert_events(acknowledged_by, occurred_at);
```

### alert_triggers
어떤 entity type + 상태 전환이 alert를 발생시키는지 정의
```sql
CREATE TABLE alert_triggers (
  id          SERIAL PRIMARY KEY,
  type_id     INTEGER NOT NULL REFERENCES entity_types(id),
  state_from  TEXT,          -- NULL = any
  state_to    TEXT NOT NULL, -- "open", "on" 등
  is_active   INTEGER DEFAULT 1
);
-- 초기값 (출입문/냉장고 열림):
-- INSERT INTO alert_triggers(type_id, state_from, state_to) VALUES (1, 'closed', 'open'); -- 출입문
-- INSERT INTO alert_triggers(type_id, state_from, state_to) VALUES (2, 'closed', 'open'); -- 냉장고
```

### monitoring_schedules
매장별 요일별 관제 시간
```sql
CREATE TABLE monitoring_schedules (
  id          SERIAL PRIMARY KEY,
  store_id    TEXT NOT NULL REFERENCES stores(store_id),
  day_of_week INTEGER NOT NULL,  -- 0=월 1=화 2=수 3=목 4=금 5=토 6=일
  start_time  TEXT NOT NULL,     -- "HH:MM"
  end_time    TEXT NOT NULL,     -- "HH:MM" — end < start 이면 익일까지
  is_active   INTEGER DEFAULT 1,
  is_manual   INTEGER DEFAULT 0, -- 0=외부서버 폴링값, 1=수동설정
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(store_id, day_of_week)
);
```
> **알림 활성 판단 우선순위**: `stores.force_alert` → `monitoring_schedules` 스케줄
> **자정 넘김**: `end_time < start_time` 이면 익일 end_time까지

### mqtt_credentials
```sql
CREATE TABLE mqtt_credentials (
  store_id      TEXT UNIQUE NOT NULL REFERENCES stores(store_id),
  username      TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,   -- bcrypt
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### users
```sql
CREATE TABLE users (
  id            SERIAL PRIMARY KEY,
  username      TEXT UNIQUE NOT NULL,  -- admin1 ~ admin5
  password_hash TEXT NOT NULL,
  display_name  TEXT
);
```

### system_config
```sql
CREATE TABLE system_config (
  key        TEXT PRIMARY KEY,
  value      TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
-- 키 목록:
-- external_server_url    외부 관제서버 API URL
-- external_server_token  인증 토큰
-- polling_interval_min   관제시간 폴링 주기 (기본: 60)
-- alert_retention_days   알림 보관 기간 (기본: 90)
-- alert_cleanup_hour     자동 삭제 시각 (기본: 14)
```

---

## 알림 활성 판단 로직 (Backend)

```
1. stores.force_alert = 1  → 무조건 알림 ON
2. stores.force_alert = 0  → 무조건 알림 OFF
3. stores.force_alert = NULL → monitoring_schedules 조회
   - 오늘 day_of_week의 start~end 범위 내인지 확인
   - end < start면 익일 end까지 포함
   - is_active = 0이면 해당 요일 관제 없음
```

## 자동 삭제 스케줄 (매일 14:00, APScheduler)

```python
DELETE FROM sensor_events WHERE occurred_at < NOW() - INTERVAL '90 days';
DELETE FROM alert_events  WHERE occurred_at < NOW() - INTERVAL '90 days';
```
