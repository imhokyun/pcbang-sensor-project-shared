# 미결 사항 (TBD)

결정되면 해당 문서 업데이트 후 이 파일에서 제거.

---

## 🔴 블로커 (착수 전 필수 결정)

현재 없음. 3개 agent 모두 착수 가능.

---

## 🟡 설계 보류 (착수 가능, 추후 반영)

### 1. 외부 관제서버 연동 포맷
**영향**: `monitoring_schedules` 자동 업데이트 로직
- URL 및 토큰: 추후 제공 예정
- 응답 포맷 미확정 (매장별 요일별 시작/종료 시간 예상)
- 폴링 주기: 기본 60분으로 구현, 추후 조정

### 2. Edge 매장 등록 — 등록 실패 처리
**영향**: HA Custom Component Config Flow UI
- 등록 실패(네트워크 오류, 잘못된 store_id) 시 사용자 안내 메시지 및 재시도 로직 상세

### 3. 알림 소리
- 중요도별로 다른 소리 필요 여부
- 브라우저 autoplay 정책: 첫 사용자 상호작용 후 활성화 필요

### 4. CSV 로그 다운로드
- `/logs` 화면 CSV export 필요 여부

### 5. 다중 카메라 알림 팝업
- 1개 알림당 1개 채널(ch01 고정?) vs 매장 전체 채널 선택 가능?

### 6. HA 기본 entity 제외 목록
- sun, zone, person, device_tracker 등 기본 제외 예정
- Edge 개발 시 테스트 후 사용자와 협의하여 제외 목록 확정
- alert_triggers 초기값도 이에 따라 추가

---

## 🟢 확정 완료

| 항목 | 결정 | 문서 |
|---|---|---|
| DB 엔진 | PostgreSQL (Docker) | db-schema.md |
| MQTT 인증 | 매장별 계정, mosquitto-go-auth + PostgreSQL | db-schema.md |
| MQTT 클라이언트 | paho-mqtt 직접 사용, HA 내장 MQTT 사용 안 함 | mqtt.md |
| MQTT keepAlive / 재연결 | keepAlive=60, clean_session=False, 지수 백오프 재연결 | mqtt.md |
| Heartbeat | 30초 publish, 90초 timeout → offline 처리 | mqtt.md |
| Entity 조회 방식 | MQTT query/response (request_id 매칭, 10초 timeout) | mqtt.md |
| Entity 상태 publish | 모니터링 목록 대상만 publish (config/monitored_entities로 수신) | mqtt.md |
| MQTT 재연결 시 monitored_entities | retain=true이므로 Mosquitto 자동 재전송 → 캐시 자동 복원 | mqtt.md |
| LWT | offline 자동 감지용 Last Will Testament 등록 | mqtt.md |
| Provisioning | Config Flow에서 store_id 입력 → secret_key로 Backend 등록 → MQTT credentials 수신 | mqtt.md |
| 알림 보관 | 기본 90일 configurable, 매일 14:00 삭제 | db-schema.md |
| 관제시간 구조 | 요일별, 자정 넘김 허용 (end<start=익일) | db-schema.md |
| 강제 알림 | force_alert (NULL=스케줄/0=강제OFF/1=강제ON) | db-schema.md |
| 매장 중요도 | 1~5, 4이상=메인화면 붉은 배경 | db-schema.md |
| go2rtc 카메라 설정 | Dashboard → Backend → go2rtc API 즉시 반영 | db-schema.md |
| DVR 채널 수 | 6~14채널, RTSP main/sub, stream_name 규칙: `{store_id}_ch{nn:02d}` | db-schema.md |
| HA Entity 등록 | Dashboard에서 HA query → 선택 → 명칭/타입 지정 | db-schema.md |
| Entity 타입 | 출입문/냉장고/카운터/기타 + 사용자 추가 | db-schema.md |
| Alert 조건 로직 | AND: triggers_alert=1 AND alert_triggers 매칭 | db-schema.md |
| 알림 활성 판단 | Backend 판단 후 WS에 is_in_schedule 포함 | api.md |
| 알림 공유 | WS 브로드캐스트, 1명 체크→전원 즉시 반영 | api.md |
| 대시보드 인증 | admin1~5 하드코딩, HttpOnly 쿠키 세션 (TTL 24h rolling) | db-schema.md, api.md |
| WS 인증 | HttpOnly 쿠키 자동 포함, 세션 없으면 연결 거부 | api.md |
| WS 재연결 | 재연결 시 서버가 init 메시지 재전송 → 전체 상태 재동기화 | api.md |
| go2rtc 접근 | Backend가 URL 전달, 방화벽 관제실 IP 허용 | architecture.md |
| 화면 구성 | login / main / stores / stores상세 / logs | frontend-screens.md |
