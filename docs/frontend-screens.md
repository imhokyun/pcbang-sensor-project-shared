# Frontend 화면 설계

## 화면 목록

```
/login                    로그인
/                         메인 관제 화면
/stores                   매장 리스트
/stores/{store_id}        매장 상세 설정
/logs                     이벤트 로그
```

---

## 1. 로그인 (`/login`)

- username / password 입력
- 로그인 성공 → `/` 리다이렉트
- 세션 만료 시 자동 리다이렉트

---

## 2. 메인 관제 화면 (`/`)

**목적**: 알림이 활성화된 매장의 실시간 이벤트를 즉시 파악

### 알림 리스트
- 미확인 alert_events를 최신순으로 표시
- 각 row: `[매장명] [센서명] [상태변화] [발생시각]`
- 중요도 4~5인 매장의 알림 row: **붉은 배경**
- 클릭 시 → 해당 매장 go2rtc 영상 팝업 + 확인 버튼
- 확인 버튼 클릭 → WS를 통해 전체 접속자에게 즉시 반영 (영상 닫힘)

### 매장 상태 요약
- 전체 매장 온라인/오프라인 현황 (카드 or 그리드)
- 각 카드: 매장명, 상태, 최근 이벤트, 중요도 배지

### 실시간 동작
- WebSocket 연결 유지
- `alert.new` 수신 → 알림 리스트 최상단에 추가 (관제시간 내인 경우에만 소리 알림)
- `alert.acknowledged` 수신 → 해당 row 즉시 확인 처리
- `entity.update` 수신 → 매장 카드 상태 갱신
- `store.status` 수신 → 온라인/오프라인 배지 갱신

---

## 3. 매장 리스트 (`/stores`)

- 전체 매장 테이블: 매장명, 주소, 온라인 상태, 중요도, 등록일
- 매장 클릭 → `/stores/{store_id}` 이동
- 매장 추가 버튼

---

## 4. 매장 상세 설정 (`/stores/{store_id}`)

### 기본 정보 탭
- 매장명, 주소, 기기 SN 수정
- 중요도 선택 (1~5 별점 or 드롭다운)
- 저장 버튼

### CCTV 설정 탭
- go2rtc URL (Edge IP) 설정
- 카메라 채널 목록 (channel 번호, 이름, RTSP main/sub URL)
- 채널 추가/수정/삭제 → 저장 즉시 Backend가 go2rtc API 호출하여 반영
- 각 채널 미리보기 버튼 (go2rtc WebRTC 스트림)

### 알림 시간 설정 탭
- 요일별 (월~일) 시작/종료 시간 입력
- 외부 관제서버 연동 표시: 마지막 폴링 시각, 현재 적용된 값
- "지금 동기화" 버튼 → `/api/v1/stores/{store_id}/schedules/sync`
- **강제 알림 설정**: 없음(스케줄) / 강제 ON / 강제 OFF (3-way toggle)

### 센서·릴레이 설정 탭
- "HA에서 Entity 가져오기" 버튼 → `/api/v1/stores/{store_id}/ha/entities` 조회
- 조회된 전체 entity 리스트 표시 (ha_entity_id, 현재 상태)
- 각 entity에 대해:
  - 등록 여부 toggle
  - 사용자 명칭 입력 (custom_name)
  - 타입 선택 (entity_types 목록, 사용자 추가 가능)
  - 알림 트리거 여부 toggle (triggers_alert)
- 저장 버튼

### 타입 관리 (센서·릴레이 탭 내 또는 별도)
- 사용자 정의 타입 추가/삭제 (기본 타입: 출입문/냉장고/카운터/기타는 삭제 불가)

---

## 5. 로그 화면 (`/logs`)

**목적**: 3개월치 상태 변화 이력 조회

### 필터
- 매장명 검색 (텍스트)
- 타입 선택 (entity_types 목록)
- 상태 변화 방향: 전체 / 열림→닫힘 / 닫힘→열림
- 기간 (from ~ to datepicker)

### 테이블 컬럼
`발생시각 | 매장명 | 센서명 | 타입 | 변화 (닫힘→열림)` 

- 페이지네이션
- CSV 다운로드 (선택사항 — TBD)

---

## 알림 소리 조건

관제시간 판단은 Frontend가 아닌 **Backend**가 판단 후 WS로 전송.
- `alert.new` 이벤트에 `is_in_schedule: true/false` 포함
- Frontend는 `is_in_schedule=true` 인 경우에만 소리 재생
- 화면 표시(리스트 추가)는 is_in_schedule 무관하게 항상
