# 관리자(Admin) 워크플로우

대시보드 계정(admin1~5)을 사용하는 관제 요원 및 시스템 관리자 기준.

---

## 1. 로그인

```
/login 접속 → username / password 입력 → 대시보드(/) 이동
```

- 세션 TTL: 24시간 rolling (요청마다 갱신)
- 세션 만료 시 자동으로 /login 리다이렉트

---

## 2. 실시간 관제 (메인 화면 `/`)

### 2-1. 알림 수신 및 확인

```
alert.new WS 수신
    ├─ 알림 리스트 최상단에 row 추가 (항상)
    ├─ is_in_schedule=true 인 경우 소리 알림 재생
    └─ 중요도 4~5인 매장 → 행 붉은 배경

알림 row 클릭
    └─ go2rtc 영상 팝업 + 확인 버튼

확인 버튼 클릭
    → POST /alerts/{alert_id}/acknowledge
    → 서버가 alert.acknowledged WS 브로드캐스트
    → 전체 접속자(5명) 화면에서 즉시 확인 처리됨
```

> 1명이 확인하면 나머지 4명 화면에도 즉시 반영. 중복 확인 불필요.

### 2-2. 매장 상태 모니터링

- 매장 카드 그리드: 온라인/오프라인 배지 실시간 갱신 (`store.status` WS)
- 센서 상태 변화: `entity.update` WS → 해당 매장 카드 즉시 갱신
- Edge 90초 무응답 → 자동 offline 표시

### 2-3. 릴레이 원격 제어

```
매장 카드 또는 매장 상세에서 릴레이 on/off 버튼 클릭
    → POST /stores/{store_id}/relays/{ha_entity_id}/command
       body: { "command": "on" | "off" }
    → Backend가 MQTT publish → Edge가 HA switch 제어
    → 결과는 entity.update WS로 자동 반영
```

---

## 3. 매장 관리 (`/stores`)

### 3-1. 새 매장 등록

```
"매장 추가" 버튼
    → POST /stores
       { store_id, name, address, device_sn, importance }
    → Edge 설치 담당자에게 store_id 전달 (installer_workflow.md 참조)
```

> store_id는 이후 변경 불가. Edge 기기와 1:1 매핑되므로 신중히 결정.

### 3-2. 매장 정보 수정 (`/stores/{store_id}` → 기본 정보 탭)

- 매장명, 주소, 기기 SN, 중요도(1~5) 수정
- 중요도 4 이상: 메인 화면 알림 row 붉은 배경 적용

### 3-3. 강제 알림 설정

| 설정값 | 동작 |
|---|---|
| 없음(스케줄) | monitoring_schedules 기준으로 알림 판단 |
| 강제 ON | 24시간 항상 알림 활성 |
| 강제 OFF | 24시간 알림 비활성 (소리 없음, 화면 표시는 유지) |

- `PUT /stores/{store_id}` body: `{ "force_alert": null | 0 | 1 }`

---

## 4. 관제 시간 설정 (`/stores/{store_id}` → 알림 시간 설정 탭)

### 4-1. 수동 시간 설정

```
요일별 시작/종료 시간 입력 → 저장
    → PUT /stores/{store_id}/schedules/{day_of_week}
       { "start_time": "22:00", "end_time": "12:00", "is_active": 1 }
    → is_manual=true 로 저장됨 (외부 서버 폴링이 덮어쓰지 않음)
```

> end_time < start_time이면 익일 end_time까지 관제 (예: 22:00~12:00 = 익일 정오까지)

### 4-2. 외부 관제서버 동기화

```
"지금 동기화" 버튼
    → POST /stores/{store_id}/schedules/sync
    → Backend가 외부 서버 즉시 폴링 → 변경 시 is_manual=false로 업데이트
```

> 외부 서버 폴링은 기본 60분 주기로 자동 실행.
> 폴링 실패 시 기존 값 유지.

---

## 5. CCTV 설정 (`/stores/{store_id}` → CCTV 설정 탭)

### 5-1. go2rtc URL 설정

- Edge IP 변경 시 go2rtc URL 업데이트: `http://{edge_ip}:1984`
- `PUT /stores/{store_id}` body에 `go2rtc_url` 포함

### 5-2. 카메라 채널 관리

```
채널 추가
    → POST /stores/{store_id}/cameras
       { "channel": 1, "name": "입구", "rtsp_main": "rtsp://...", "rtsp_sub": "rtsp://..." }
    → Backend가 go2rtc API 즉시 호출하여 스트림 등록
    → stream_name: {store_id}_ch{nn:02d} (예: store_001_ch01)

채널 미리보기 버튼
    → go2rtc WebRTC 스트림 직접 재생 (브라우저 ↔ Edge 직접 연결)
```

> 카메라 저장/수정 즉시 go2rtc에 반영됨. 별도 재시작 불필요.

---

## 6. 센서·릴레이 설정 (`/stores/{store_id}` → 센서·릴레이 탭)

### 6-1. HA Entity 조회

```
"HA에서 Entity 가져오기" 버튼
    → GET /stores/{store_id}/ha/entities
    → Backend가 MQTT query → Edge 응답 대기 (최대 10초)
    → HA 기본 entity 제외한 전체 목록 표시
```

> Edge가 오프라인이거나 10초 응답 없으면 503 오류.

### 6-2. Entity 등록 및 설정

각 entity에 대해:

| 항목 | 설명 |
|---|---|
| 등록 여부 | toggle — 등록 시 monitored_entities 목록에 포함 |
| 사용자 명칭 | custom_name (예: "출입구 도어") |
| 타입 | entity_types 목록에서 선택 (출입문/냉장고/카운터/기타 또는 사용자 추가) |
| 알림 트리거 | triggers_alert toggle — 1이면 alert_triggers 매칭 시 알림 발생 |

```
저장
    → POST/PUT /stores/{store_id}/entities
    → Backend가 monitored_entities MQTT 재발행
    → Edge가 retain 메시지 수신하여 모니터링 목록 자동 업데이트
```

### 6-3. Alert 발생 조건

alert는 **두 조건 모두** 충족할 때 발생:
1. `store_entities.triggers_alert = 1` (entity가 알림 대상으로 등록)
2. `alert_triggers` 테이블에 해당 타입의 state 전환 조건 매칭

> 예: 출입문 타입 entity, triggers_alert=1, closed→open 전환 → alert 발생

---

## 7. 이벤트 로그 조회 (`/logs`)

```
필터: 매장명 | 타입 | 상태변화 방향 | 기간(from~to)
결과: 발생시각 | 매장명 | 센서명 | 타입 | 변화 방향
페이지네이션: page / limit (기본 50)
```

- 3개월치 이력 조회 가능 (보관 기간: system_config `alert_retention_days`, 기본 90일)

---

## 8. 시스템 설정

```
GET  /config          → 현재 설정 조회
PUT  /config          → 설정 수정
  { "external_server_url": "...", "external_server_token": "...",
    "polling_interval_min": 60, "alert_retention_days": 90 }
```

---

## 알림 판단 흐름 요약

```
센서 상태 변화
    │
    ├─ store_entities.triggers_alert = 0 → 무시
    │
    └─ triggers_alert = 1
           │
           └─ alert_triggers 테이블 매칭 여부 확인
                  │
                  ├─ 매칭 없음 → sensor_events만 저장
                  │
                  └─ 매칭 있음 → alert_events 저장
                                     │
                                     └─ 알림 활성 판단
                                            ├─ force_alert=1 → is_in_schedule=true
                                            ├─ force_alert=0 → is_in_schedule=false
                                            └─ NULL → monitoring_schedules 조회
                                                WS alert.new 브로드캐스트
```
