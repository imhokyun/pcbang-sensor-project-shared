# 무인매장 관제 서비스 — 프로젝트 비전

## 서비스 목표

다수의 무인매장에 설치된 센서·릴레이·카메라를 중앙 관제 대시보드에서 실시간으로 모니터링하고 제어한다.
센서 이벤트 발생 시 해당 매장의 CCTV 영상을 즉시 확인하고, 관제 시간대에 따라 선택적으로 알림을 제공한다.

- 초기 규모: 10개 매장 / 최대 100개 매장
- 관제 운영 시간: 22:00 ~ 익일 12:00 (매장별·요일별 상이)

---

## 구성 컴포넌트

### Edge (매장 — Raspberry Pi 4)
| 소프트웨어 | 역할 |
|---|---|
| Home Assistant OS | GPIO 센서/릴레이 통합, 자동화 엔진 |
| HA Custom Component | 우리 서비스 전용 센서/릴레이 MQTT 연동 및 자동 등록 |
| go2rtc | DVR RTSP 스트림 → WebRTC/HLS 변환 (브라우저 직접 재생) |

### Backend (중앙 서버)
| 소프트웨어 | 역할 |
|---|---|
| Mosquitto + mosquitto-go-auth | 중앙 MQTT 브로커, 매장별 계정 인증 (SQLite 조회) |
| FastAPI (Python) | REST API + WebSocket 서버 + 스케줄러 |
| SQLite (WAL) | 단일 DB 파일, Backend 전용 writer |

### Frontend (관제 대시보드 — Next.js)
| 기능 | 설명 |
|---|---|
| 매장 목록 | 전체 매장 센서/릴레이 상태 한눈에 보기 |
| 실시간 업데이트 | WebSocket으로 센서 변화 즉시 반영 |
| 알림 리스트 | 이벤트 트리거 시 최상단에 추가, 영상 팝업 |
| 알림 확인 공유 | 1명 체크 → 전체 5명 계정에 즉시 반영 |
| 릴레이 제어 | 대시보드에서 매장 릴레이 원격 on/off |

---

## 데이터 흐름

```
[RP4 GPIO 센서/릴레이]
        ↕ HA Custom Component
[Home Assistant OS]
        ↓ MQTT publish  (topic: pcbang/{store_id}/sensors/{sensor_id}/state)
[중앙 Mosquitto — 매장별 계정 인증]
        ↓ subscribe
[FastAPI Backend]
        ├─ sensor_events / alert_events DB 저장
        ├─ alert_triggers 조회 → 알림 여부 판단
        ├─ monitoring_schedules 조회 → 관제시간 내 여부 판단
        └─ WebSocket broadcast → [Next.js 대시보드 (5명)]

[DVR — RTSP]
        ↓
[go2rtc (Edge, 매장 내)]
        ↓ WebRTC / HLS  (Backend가 URL 전달, 방화벽에서 관제실 IP만 허용)
[Frontend 브라우저 — 직접 연결]
```

---

## 핵심 알림 시나리오

1. 센서 이벤트 발생 → Backend가 `alert_triggers` 테이블 조회
2. 트리거 조건 매칭 → `alert_events` 저장
3. `monitoring_schedules` 조회 → 현재 관제시간 내인지 판단
4. 관제시간 내: WebSocket으로 전체 클라이언트에 알림 + 영상 URL 전송
5. 관제시간 외: DB 저장만, 알림 없음
6. 대시보드에서 알림 체크 → DB 업데이트 → WS 브로드캐스트 → 전원 체크 반영

---

## MQTT Topic 구조

```
pcbang/{store_id}/sensors/{sensor_id}/state      # 센서 상태
pcbang/{store_id}/sensors/{sensor_id}/attributes # 센서 메타정보
pcbang/{store_id}/relays/{relay_id}/state        # 릴레이 현재 상태
pcbang/{store_id}/relays/{relay_id}/set          # 릴레이 제어 명령 (Backend → Edge)
pcbang/{store_id}/status                         # Edge heartbeat (online/offline)
```

---

## 관제시간 관리

- 외부 관제서버에서 매장별·요일별 시작/종료 시간 폴링
- 변경 감지 시 자동 업데이트 (`is_manual=false`)
- 폴링 실패 시 기존값 유지, 수동 설정 가능 (`is_manual=true`)
- 자정 넘김 허용: `end_time < start_time` → 익일 end_time까지

---

## 인증

| 대상 | 방식 |
|---|---|
| MQTT (Edge ↔ Broker) | 매장별 username/password, mosquitto-go-auth + SQLite |
| 대시보드 | admin1~5 하드코딩 계정, 세션 기반 |
| Backend API | (내부 전용, 별도 인증 없음 또는 심플 토큰) |

---

## 미결 사항

상세 내용은 `docs/tbd.md` 참조.
- Edge 기기 구성 및 HA Entity 전체 목록
- 알림 트리거 조건 전체 목록
- 외부 관제서버 API 포맷 (URL/토큰 추후 제공)
- 매장별 DVR 카메라 수 및 go2rtc stream 명칭 규칙
