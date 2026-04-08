# 사용자 피드백 반영 — 전체 작업 계획

**발행일**: 2026-04-08  
**작성**: Orchestrator  
**상태**: 팀별 작업 지시 발행 완료

---

## 배경

실 사용자 피드백을 기반으로 메인 관제 알림페이지와 매장 관리 화면을 개선한다.

---

## 전체 작업 지도

```
┌─────────────────────────────────────────────────────────────────────┐
│                        이번 스프린트 목표                            │
├─────────────────────────────────────────────────────────────────────┤
│  1. 알림이 발생했을 때 관제 담당자가 상황을 빠르게 파악할 수 있도록  │
│     알림 팝업 UI 개선 (매장명·센서명·시간 강조)                       │
│                                                                     │
│  2. 알림 목록이 기기 타입별로 구분되어 보여야 함                     │
│     (냉장고/출입문/금고 섹션 분리)                                   │
│                                                                     │
│  3. 중요도 4·5 알림은 소리로도 긴박함을 전달해야 함                  │
│                                                                     │
│  4. 센서가 오래 열려있을 때 자동 알림                                │
│     (닫히지 않은 상태 감지 → 알림 생성)                              │
│                                                                     │
│  5. 매장이 많아질 때를 대비한 검색 + 페이지네이션                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 작업 항목별 담당 팀

| ID | 작업 | 담당 | 의존성 |
|----|------|------|--------|
| A1 | 알림 팝업 UI — 매장명·센서명·시간 강조, 카메라 작게 | Frontend | 없음 |
| A2 | 알림 목록 — 기기 타입별 섹션 분리 | Frontend | 없음 |
| A3 | 알림음 — 중요도 4·5 강조 소리 | Frontend | 없음 |
| A4-BE | 장시간 개방 감지 — 백그라운드 태스크 + 알림 생성 | Backend | 없음 |
| A4-FE | 장시간 개방 알림 표시 | Frontend | A4-BE 완료 후 |
| A4-Edge | HA 자동화 보완 (선택) | Edge | A4-BE 완료 후 |
| B1-BE | 매장 API — 검색·페이지네이션 쿼리 파라미터 추가 | Backend | 없음 |
| B1-FE | 매장 화면 — 검색 UI + 페이지네이션 컴포넌트 | Frontend | B1-BE 완료 후 (UI 골격은 선행 가능) |

---

## 병렬 실행 계획

```
[즉시 시작 가능]
  Lane A — Frontend: A3 → A1 → A2 (같은 파일 건드리므로 순차)
  Lane B — Backend:  A4-BE (long_open 태스크) / B1-BE (stores API)

[Lane A·B 완료 후]
  Lane C — Frontend: A4-FE, B1-FE
  Lane D — Edge:     A4-Edge (선택)
```

---

## A4 — 장시간 개방 알림 전체 플로우

```
[HA / Edge]                [Backend]                  [Frontend]
  센서 state_to='on'  ──MQTT──▶  SensorEvent 저장
                                       │
                         (1분 주기 asyncio task)
                                       │
                         SensorEvent에서 마지막 'on' 후
                         threshold_minutes 초과 여부 검사
                                       │ 초과 시
                         AlertEvent 생성 (type='장시간개방')
                                       │
                                  WebSocket ──────────▶ 기존 알림 표시
                                                         (추가 개발 없음)

임계값 저장 위치: EntityType.threshold_minutes (타입별 설정)
예시: 냉장고=5분, 출입문=10분, 금고=2분
```

---

## B1 — 매장 검색·페이지네이션 플로우

```
[Frontend]                          [Backend]
  검색어 입력 (debounce 300ms)
          │
          └──GET /stores?q=강남&page=1&limit=100──▶  DB WHERE name ILIKE '%강남%'
                                                        OR address ILIKE '%강남%'
                                                        LIMIT 100 OFFSET 0
                                                              │
  ◀──{ items: [...], total: N, page: 1, total_pages: M }──────┘
          │
  페이지네이션 UI 렌더링
```

---

## 각 팀 작업 지시 파일 위치

- Frontend: `shared/to_frontend/from_orchestrator_20260408_tasks.md`
- Backend: `shared/to_backend/from_orchestrator_20260408_tasks.md`
- Edge: `shared/to_edge/from_orchestrator_20260408_tasks.md`

---

## 완료 기준

| ID | 완료 기준 |
|----|-----------|
| A1 | 알림 팝업에서 매장명·센서명·시간이 카메라보다 시각적으로 두드러짐 |
| A2 | 기기 타입별 섹션 헤더 표시, 빈 섹션 숨김 |
| A3 | importance 4·5 알림 시 기존과 다른 소리 재생 |
| A4 | 임계 시간 초과 센서에 대해 자동 AlertEvent 생성 및 WebSocket 전송 |
| B1 | `/stores?q=검색어` 로 DB 필터링, 100개 페이지네이션 동작 |
