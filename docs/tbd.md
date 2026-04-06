# 미결 사항 (TBD)

개발 착수 전 확정이 필요한 항목들. 결정되면 해당 문서를 업데이트하고 이 파일에서 제거.

---

## 🔴 블로커 (착수 전 필수 결정)

### 1. Edge 기기 구성 — HA Entity 전체 목록
**영향**: `alert_triggers` 초기값, `sensors` 타입 확정, HA Custom Component 설계
- 어떤 GPIO 센서/릴레이를 연결하는지 (도어, 냉장고, motion, 온도, 기타)
- HA에서 어떤 entity_id 형식으로 노출할지
- 릴레이 몇 개, 어떤 용도인지

### 2. 알림 트리거 조건 전체 목록
**영향**: `alert_triggers` 테이블, Backend MQTT handler 로직
- 현재 확정: `door` closed→open, `refrigerator_door` closed→open
- 추가 트리거 대상: motion? 온도 임계값? 기타?
- 임계값 기반 알림(temperature > 30°C 등) 필요하면 `alert_triggers`에 `threshold` 컬럼 추가 필요

---

## 🟡 설계 보류 (착수는 가능하나 추후 반영 필요)

### 3. 외부 관제서버 연동
**영향**: `system_config`, polling 서비스, `monitoring_schedules` 자동 업데이트
- URL 및 토큰: 추후 제공 예정
- 응답 데이터 포맷: 미확정 (매장별 요일별 시간 정보)
- 폴링 주기: 미정 (기본값 60분으로 구현 후 조정)

### 4. Edge 매장 등록 provisioning 절차 상세
**영향**: HA Custom Component 초기 설정 화면, Backend `/register` API
- RP4 이미지 굽기 시 `store_id` + `secret_key` 심는 방식 상세
- 최초 부팅 → Backend 등록 요청 → 외부 관제서버 매칭 → MQTT 계정 발급 흐름 구체화

### 5. go2rtc 스트림 URL 구조
**영향**: `stores.go2rtc_url` 저장 방식, Frontend 영상 팝업 URL 생성
- 매장별 DVR 카메라 수: 1개? 여러 개?
- 여러 개라면 `streams` 별도 테이블 필요 여부
- go2rtc stream name 규칙 (예: `cam1`, `entrance`, ...)

### 6. 관제시간 외 알림 처리
**영향**: Backend 알림 판단 로직
- 관제시간 외에 발생한 이벤트: DB에는 기록하되 WS 알림 안 보내는 것으로 설계 예정
- 관제시간 외 이벤트를 나중에 확인할 수 있어야 하는지?

---

## 🟢 확정 완료 (참고용)

| 항목 | 결정 내용 | 문서 |
|---|---|---|
| DB 엔진 | SQLite WAL, 단일 파일 | db-schema.md |
| MQTT 인증 | 매장별 계정, mosquitto-go-auth + SQLite | db-schema.md |
| 알림 보관 | 3개월, 매일 14:00 자동 삭제 | db-schema.md |
| 관제시간 구조 | 요일별, 자정 넘김 허용 (end<start=익일) | db-schema.md |
| go2rtc 접근 | Backend가 URL 전달, 브라우저 직접 연결, 방화벽 관제실 IP 허용 | architecture.md |
| 알림 공유 | WS 브로드캐스트, 1명 체크→전원 반영 | api.md |
| 대시보드 인증 | admin1~5 하드코딩 멀티계정 | db-schema.md |
| 관제시간 폴링 실패 | 마지막 값 유지 + 수동 설정 가능 | db-schema.md |
| 알림 트리거 방식 | alert_triggers 테이블로 관리 | db-schema.md |
