# 무인매장 관제 서비스 — 프로젝트 비전

## 서비스 목표

다수의 무인매장에 설치된 센서·릴레이·카메라를 중앙 관제 대시보드에서 실시간으로 모니터링하고 제어한다.
센서 이벤트(출입구 열림, 냉장고 문 열림 등) 발생 시 해당 매장의 CCTV 영상을 즉시 확인할 수 있다.

---

## 구성 컴포넌트

### Edge (매장 — Raspberry Pi 4)
| 소프트웨어 | 역할 |
|---|---|
| Home Assistant OS | GPIO 센서/릴레이 통합, 자동화 엔진 |
| HA Custom Component | 우리 서비스 전용 센서/릴레이 등록 및 MQTT 연동 |
| go2rtc | DVR RTSP 스트림 → WebRTC/HLS 변환 (브라우저 직접 재생) |
| Mosquitto (로컬) | 로컬 MQTT 브로커 (HA ↔ 중앙 브릿지용, 선택) |

### Backend (중앙 서버)
| 소프트웨어 | 역할 |
|---|---|
| Mosquitto | 중앙 MQTT 브로커 — 모든 매장 Edge가 여기로 publish |
| FastAPI (Python) | REST API + WebSocket 서버 |
| DB | 매장/센서/이벤트 이력 저장 |

### Frontend (관제 대시보드 — Next.js)
| 기능 | 설명 |
|---|---|
| 매장 목록 | 전체 매장 센서 상태 한눈에 보기 |
| 실시간 업데이트 | WebSocket으로 센서 변화 즉시 반영 |
| 영상 팝업 | 이벤트 트리거 시 해당 매장 go2rtc 스트림 자동 표시 |
| 릴레이 제어 | 대시보드에서 매장 릴레이 원격 on/off |

---

## 데이터 흐름

```
[RP4 GPIO 센서/릴레이]
        ↕ HA Integration
[Home Assistant OS]
        ↓ MQTT publish  (topic: pcbang/{store_id}/sensors/{sensor_id})
[중앙 Mosquitto Broker]
        ↓ subscribe
[FastAPI Backend]
        ├─ DB 저장 (이벤트 이력)
        └─ WebSocket broadcast → [Next.js Frontend]

[DVR — RTSP]
        ↓ RTSP stream
[go2rtc (Edge)]
        ↓ WebRTC / HLS
[Frontend 브라우저 — 이벤트 발생 시 팝업]
```

---

## 핵심 이벤트 시나리오

| 트리거 | 동작 |
|---|---|
| 출입구 센서 닫힘→열림 | 해당 매장 영상 스트림 팝업 + 이벤트 이력 저장 |
| 냉장고 문 닫힘→열림 | 해당 매장 영상 스트림 팝업 + 이벤트 이력 저장 |
| 릴레이 상태 변경 | 대시보드 상태 즉시 업데이트 |
| 센서 오프라인 감지 | 대시보드 경고 표시 |

---

## MQTT Topic 구조

```
pcbang/{store_id}/sensors/{sensor_id}/state      # 센서 상태 (on/off/value)
pcbang/{store_id}/sensors/{sensor_id}/attributes # 센서 메타정보
pcbang/{store_id}/relays/{relay_id}/state        # 릴레이 상태
pcbang/{store_id}/relays/{relay_id}/set          # 릴레이 제어 명령 (backend → edge)
pcbang/{store_id}/status                         # Edge 온라인/오프라인 heartbeat
```

---

## 미결 사항 (논의 필요)

- [ ] DB 선택: PostgreSQL vs TimescaleDB (시계열 이벤트 이력)
- [ ] 센서 타입 확정 목록
- [ ] go2rtc 접근 방식: Edge 직접 접근 vs Backend 프록시
- [ ] 인증/보안: MQTT 인증, API 토큰, 대시보드 로그인
- [ ] 매장 수 규모 (동시 연결 Edge 수)
- [ ] 알림: 이벤트 발생 시 푸시/문자 알림 필요 여부
