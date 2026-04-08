# 사용자 피드백 반영 — 전체 작업 계획 (Rev.2)

**발행일**: 2026-04-08 (Rev.2 수정)  
**작성**: Orchestrator  
**변경 이유**: 각 팀 검토 의견 반영 + 코드베이스 실제 상태 기준 재정리

> **Rev.1 대비 주요 변경**: A4 아키텍처 변경 승인 (Backend 폴링 → Edge 감지), A4-Edge 필수로 격상

---

## 확정 결정 사항

### A4 아키텍처: Edge 감지 방식 채택

Backend·Edge 양 팀의 건의를 검토한 결과, **Edge 감지 + Backend MQTT 수신** 방식으로 변경합니다.

| 항목 | 기존 계획 | 변경 후 |
|------|-----------|---------|
| 감지 주체 | Backend (DB 1분 폴링) | Edge (asyncio 타이머) |
| 감지 정확도 | ±1분 | 초 단위 정밀 |
| Backend 작업 | DB 마이그레이션 + 폴링 서비스 | MQTT 구독 추가 + AlertEvent 생성 |
| Edge 작업 | Optional | **필수 (이번 스프린트)** |
| threshold 저장 | EntityType.threshold_minutes (DB) | DB에 저장 후 monitored_entities로 Edge에 전달 |

**단, EntityType.threshold_minutes 컬럼은 여전히 필요** — Backend가 threshold 값을 Edge에 전달해야 하므로.

### A1 팝업 UI: custom_name 사용 확정

Frontend 팀 확인: `entity_name` 필드는 현재 Alert 타입에 없음. `custom_name` 사용.  
(Backend는 WS에 `entity_name`도 보내지만 Frontend는 `custom_name`을 사용하는 것이 코드 일관성상 맞음.)

---

## 전체 작업 지도

```
┌─────────────────────────────────────────────────────────────────────┐
│                        이번 스프린트 목표                            │
├─────────────────────────────────────────────────────────────────────┤
│  1. 알림 팝업 UI 개선 (매장명·센서명·시간 강조, 카메라 작게)          │
│  2. 알림 목록 기기 타입별 섹션 분리                                   │
│  3. 중요도 4·5 알림 강조 소리                                        │
│  4. 센서 장시간 개방 자동 알림 (Edge asyncio 타이머 방식)             │
│  5. 매장 검색 + 페이지네이션                                          │
└─────────────────────────────────────────────────────────────────────┘
```

## 작업 항목별 담당 팀 (Rev.2)

| ID | 작업 | 담당 | 의존성 | 변경여부 |
|----|------|------|--------|---------|
| A1 | 알림 팝업 UI — 매장명·센서명·시간 강조, 카메라 작게 | Frontend | 없음 | custom_name 확정 |
| A2 | 알림 목록 — 기기 타입별 섹션 분리 | Frontend | 없음 | 동일 |
| A3 | 알림음 — 중요도 4·5 강조 소리 | Frontend | 없음 | 동일 |
| A4-BE | 장시간 개방 — MQTT 구독 + AlertEvent 생성 | Backend | A4-Edge 완료 후 테스트 | **방식 변경** |
| A4-Edge | 장시간 개방 — asyncio 타이머 + MQTT 발행 | Edge | A4-BE 완료 후 연동 | **필수로 격상** |
| A4-FE | 장시간 개방 알림 표시 확인 | Frontend | A4-BE + A4-Edge 완료 후 | 동일 |
| B1-BE | 매장 API — 검색·페이지네이션 | Backend | 없음 | 동일 |
| B1-FE | 매장 화면 — 검색 UI + 페이지네이션 | Frontend | B1-BE 완료 후 (UI 선행 가능) | 동일 |

---

## A4 전체 플로우 (Rev.2)

```
[Edge / HA]                        [Backend]                 [Frontend]
  sensor state → on
  asyncio 타이머 시작 (threshold_minutes)
          │ 타이머 만료 (초 단위 정밀)
          └──MQTT publish──────────▶ pcbang/{store_id}/entities/{ha_entity_id}/long_open
                                            │
                                   AlertEvent 생성
                                   (type_name='장시간개방',
                                    중복 방지 체크)
                                            │
                                   WS broadcast ──────────▶ 기존 alert 표시 로직
                                                             (A2 섹션에 자동 분류)

threshold 전달 경로:
  Backend DB (EntityType.threshold_minutes)
  → monitored_entities MQTT payload에 포함 (entities: object[])
  → Edge가 수신 후 타이머에 적용
```

---

## B1 매장 검색·페이지네이션 플로우 (동일)

```
[Frontend]                          [Backend]
  검색어 입력 (debounce 300ms)
          │
          └──GET /stores?q=강남&page=1&limit=100──▶  DB WHERE name/address ILIKE
                                                              │
  ◀──{ items: [...], total: N, page: 1, total_pages: M }──────┘
          │
  페이지네이션 UI 렌더링
```

---

## 병렬 실행 계획 (Rev.2)

```
[즉시 시작 가능]
  Lane A — Frontend:  A3 → A1 → A2 (순차, 같은 파일 영향)
  Lane A' — Frontend: B1-FE UI 골격 (A와 별도, page.tsx 파일)
  Lane B — Backend:   A4-BE (MQTT 구독 + EntityType 마이그레이션) + B1-BE (병렬)
  Lane C — Edge:      A4-Edge (coordinator.py 타이머 구현)

[Lane B·C 연동 후]
  A4 통합 테스트: Edge 타이머 발화 → Backend AlertEvent → Frontend 수신 확인
  B1-FE API 연결: Backend B1-BE 완료 통보 후
```

---

## 완료 기준

| ID | 완료 기준 |
|----|-----------|
| A1 | 팝업에서 custom_name(센서명)·store_name·시각이 카메라보다 시각적으로 두드러짐 |
| A2 | type_name별 섹션 헤더, 빈 섹션 숨김, '장시간개방' 섹션 자동 생성 |
| A3 | importance ≥ 4 알림 시 기존과 다른 소리 재생 |
| A4 | Edge 타이머 만료 → Backend AlertEvent 생성 → Frontend 알림 수신 확인 |
| B1 | `/stores?q=검색어&page=1&limit=100` DB 필터링 + Frontend 검색/페이지네이션 동작 |

---

## 각 팀 작업 지시 파일 위치

- Frontend: `shared/to_frontend/from_orchestrator_20260408_tasks_rev2.md`
- Backend: `shared/to_backend/from_orchestrator_20260408_tasks_rev2.md`
- Edge: `shared/to_edge/from_orchestrator_20260408_tasks_rev2.md`
