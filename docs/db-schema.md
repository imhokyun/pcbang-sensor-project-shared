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
  store_id        INTEGER UNIQUE NOT NULL, -- 관제서버 storeNo (예: 30584)
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
  id             SERIAL PRIMARY KEY,
  store_id       INTEGER NOT NULL REFERENCES stores(store_id),
  channel        INTEGER NOT NULL,      -- 1~14
  name           TEXT,                  -- "입구", "카운터" (사용자 지정)
  stream_source  TEXT,                  -- go2rtc 소스명: "ch1_sub", "ch2_sub"
  is_active      INTEGER DEFAULT 1,
  UNIQUE(store_id, channel)
);
```
> **stream_url 조합**: `stores.go2rtc_url + "/stream.html?src=" + cameras.stream_source`  
> go2rtc 자체 설정(RTSP 주소 등)은 Edge 팀이 go2rtc config.yaml로 관리. Backend는 소스명만 저장.

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
  store_id       INTEGER NOT NULL REFERENCES stores(store_id),
  ha_entity_id   TEXT NOT NULL,       -- HA entity_id: "binary_sensor.door_01"
  entity_kind    TEXT NOT NULL,       -- sensor / switch
  custom_name    TEXT,                -- "출입구 도어" (사용자 지정 명칭)
  type_id        INTEGER REFERENCES entity_types(id),
  triggers_alert INTEGER DEFAULT 0,   -- 1=상태변화 시 alert 발생 대상
  camera_channel INTEGER,             -- 연결된 카메라 채널 번호 (NULL=미연결)
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
  store_id     INTEGER NOT NULL,
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
  acknowledged_by INTEGER DEFAULT NULL REFERENCES users(id),  -- NULL = 미확인
  acknowledged_at TIMESTAMPTZ DEFAULT NULL,                   -- NULL = 미확인
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
  store_id      INTEGER NOT NULL REFERENCES stores(store_id),
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

### sessions
HttpOnly 쿠키 세션 저장소
```sql
CREATE TABLE sessions (
  id         TEXT PRIMARY KEY,         -- UUID v4
  user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,     -- created_at + 24h (rolling 갱신)
  last_seen_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
```
> 세션 TTL: 24시간 rolling. 매 요청마다 `last_seen_at` 갱신, `expires_at` 연장.
> 만료 세션 자동 삭제: `DELETE FROM sessions WHERE expires_at < NOW()` (APScheduler, 매일 02:00)

### system_config
```sql
CREATE TABLE system_config (
  key        TEXT PRIMARY KEY,
  value      TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
-- 키 목록:
-- external_server_url     외부 관제서버 API URL
-- external_server_token   인증 토큰
-- polling_interval_min    관제시간 폴링 주기 (기본: 60)
-- alert_retention_days    알림/센서 이벤트 보관 기간 (기본: 90)
-- alert_cleanup_hour      자동 삭제 시각 (기본: 14)
```

---

## 알림 발생 조건 (Backend)

alert 발생은 아래 **두 조건 모두** 충족해야 한다 (AND):

```
조건 1. store_entities.triggers_alert = 1  (해당 entity가 알림 대상으로 등록됨)
조건 2. alert_triggers 테이블에 매칭되는 행 존재
        - type_id = 해당 entity의 type_id
        - state_from = NULL (any) 또는 이전 상태와 일치
        - state_to = 현재 상태와 일치
        - is_active = 1
```

alert_events 저장 후 알림 활성 여부 판단:

```
1. stores.force_alert = 1  → is_in_schedule = true
2. stores.force_alert = 0  → is_in_schedule = false
3. stores.force_alert = NULL → monitoring_schedules 조회
   - 오늘 day_of_week의 start~end 범위 내인지 확인
   - end < start면 익일 end까지 포함
   - is_active = 0이면 해당 요일 관제 없음 → is_in_schedule = false
```

결과를 `alert.new` WebSocket 메시지의 `is_in_schedule` 필드에 포함하여 브로드캐스트.

## 자동 삭제 스케줄 (매일 14:00, APScheduler)

보관 기간은 `system_config` 테이블의 `alert_retention_days` 값 사용 (기본 90일).

```python
retention = get_config("alert_retention_days", default=90)
DELETE FROM sensor_events WHERE occurred_at < NOW() - INTERVAL '{retention} days';
DELETE FROM alert_events  WHERE occurred_at < NOW() - INTERVAL '{retention} days';
DELETE FROM sessions      WHERE expires_at  < NOW();
```
