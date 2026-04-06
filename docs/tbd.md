# 미결 사항 (TBD)

결정되면 해당 문서 업데이트 후 이 파일에서 제거.

---

## 🔴 블로커 (착수 전 필수 결정)

### 1. HA Entity 전체 목록 및 MQTT topic 규칙
**영향**: HA Custom Component 설계, `store_entities` 등록 flow
- 기본 구성: switch 2개, sensor 4개 (타입 미확정)
- HA에서 외부로 entity 상태를 MQTT에 publish하는 방식:
  - HA 내장 MQTT integration 사용 (autodiscovery) vs Custom Component에서 직접 publish
  - **추천**: Custom Component에서 직접 `pcbang/{store_id}/entities/{ha_entity_id}/state` 발행

### 2. HA → Backend query_all_entity 통신 방식
**영향**: `/api/v1/stores/{store_id}/ha/entities` 구현
- Backend가 HA에 entity 목록을 어떻게 요청하나?
  - Option A: MQTT request/response (`pcbang/{store_id}/query/entities` 발행 → HA가 응답)
  - Option B: HA REST API 직접 호출 (HA Long-lived token 필요, 포트 개방 필요)
  - **추천**: Option A (MQTT로 통일, 포트 추가 개방 불필요)

---

## 🟡 설계 보류 (착수 가능, 추후 반영)

### 3. 외부 관제서버 연동 포맷
**영향**: `monitoring_schedules` 자동 업데이트 로직
- URL 및 토큰: 추후 제공
- 응답 포맷 미확정 (매장별 요일별 시작/종료 시간 예상)
- 폴링 주기: 기본 60분으로 구현, 추후 조정

### 4. Edge 매장 등록 Provisioning 상세
**영향**: HA Custom Component 초기 설정 UI, `/edge/register` API
- RP4 이미지에 store_id + secret_key 심는 방법 (USB? 이미지 빌드 시 포함?)
- 등록 실패 시 재시도 로직 및 사용자 안내

### 5. go2rtc stream 이름 규칙 확정
현재 안: `{store_id}_ch{channel:02d}` (예: `store_001_ch01`)
- 변경 시 `cameras.stream_name` 컬럼 및 API URL에 영향

### 6. 알림 소리 파일
- 중요도별로 다른 소리 필요 여부
- 브라우저 autoplay 정책 우회 방법 (첫 상호작용 후 활성화)

### 7. CSV 로그 다운로드
- `/logs` 화면에서 CSV export 필요 여부

### 8. 다중 카메라 알림 시 팝업 처리
- 1개 알림당 1개 카메라 팝업 or 매장 전체 채널 선택 가능하게?

---

## 🟢 확정 완료

| 항목 | 결정 | 문서 |
|---|---|---|
| DB 엔진 | SQLite WAL | db-schema.md |
| MQTT 인증 | 매장별 계정, mosquitto-go-auth | db-schema.md |
| 알림 보관 | 3개월, 매일 14:00 삭제 | db-schema.md |
| 관제시간 구조 | 요일별, 자정 넘김 허용 | db-schema.md |
| 강제 알림 | force_alert (NULL/0/1) | db-schema.md |
| 매장 중요도 | 1~5, 4이상=메인화면 붉은 배경 | db-schema.md |
| go2rtc 카메라 | Dashboard에서 설정 → go2rtc API 즉시 반영 | db-schema.md |
| DVR 채널 수 | 6~14채널, RTSP main/sub | db-schema.md |
| HA Entity 등록 | Dashboard에서 HA query → 선택 → 명칭/타입 지정 | db-schema.md |
| Entity 타입 | 출입문/냉장고/카운터/기타 + 사용자 추가 | db-schema.md |
| 알림 활성 판단 | Backend 판단 후 WS에 is_in_schedule 포함 | api.md |
| 알림 공유 | WS 브로드캐스트, 1명 체크→전원 반영 | api.md |
| go2rtc 접근 | Backend가 URL 전달, 방화벽 관제실 IP 허용 | architecture.md |
| 대시보드 인증 | admin1~5 하드코딩 | db-schema.md |
| 화면 구성 | login/main/stores/stores상세/logs | frontend-screens.md |
