# 2026-05-26 UX 스프린트 — 실행 계획 초안

원본 요구사항: [`shared/20260526_dev.md`](../20260526_dev.md)
검토 단계: **`/plan-eng-review` 진행 중 (Scope: Phase 1 락인 대상, Phase 2는 후속 스프린트)**
오케스트레이터: orchestrator (main 워크트리)

## Phase 분할 (Step 0 — Scope Challenge 결과)

복잡도 게이트 트리거 (20+ files, 5+ 신규 모델·라우터·컴포넌트) → **두 Phase로 분할**.

| Phase | Task | 비고 |
|---|---|---|
| **Phase 1 — 즉시 UX 효과** | T1 시인성, T2 카테고리 필터, T3 중요도, T4a 온·오프라인, T6 일괄확인 | 이번 스프린트. DB 모델 변경 없음. WS 페이로드 확장만. |
| **Phase 2 — 보관 워크플로우** | T4b 보관(처리중) + 메모 + 완료 + 보관로그 | 다음 스프린트. 신규 `alert_archives` 테이블. Phase 1 운영 피드백 후 진행. |
| ~~T5 CTRL 양방향 연동~~ | **이번 스프린트 제외** | 사용자 명시 제외. 외부 시스템 정합성 협의 선행 필요. NOT in scope 섹션에 별도 정리. |

**이 review의 락인 대상은 Phase 1.** Phase 2 항목은 1 완료 후 별도 `/plan-eng-review` 라운드를 거친다.

이 문서는 plan-eng-review 가 architecture / data flow / edge case / test / performance 관점에서 review 할 수 있게 작성한 **실행 계획 초안**이다. 합의 후 각 task를 backend / frontend 워크트리의 teammate agent에게 분배한다.

---

## 0. 스프린트 목표

운영서버(192.168.0.70) 이관 직후 실사용 피드백 기반 UX 개선 + 일괄확인 / 외부서버(CTRL) 양방향 연동.

- 가시성·정보밀도 개선 (시인성 토픽 1·3)
- 카테고리 라우팅 (토픽 2)
- 매장 운영 보조 기능 (토픽 4 — 보관/메모/완료 워크플로우)
- CTRL서버 양방향 동기화 (토픽 5)
- 다중 알림 처리 (토픽 6 — 일괄확인)

---

## 1. 작업 항목 (요구사항 → task)

### T1. 메인관제 카드·모달 시인성 (Frontend only)

- **변경 대상**
  - `frontend/components/StoreGrid.tsx` (대시보드 카드)
  - `frontend/components/AlertList.tsx` (좌측 알림 리스트)
  - `frontend/components/VideoPopup.tsx` (모달)
  - `frontend/DESIGN.md` (Decisions Log 갱신)
- **결정사항**
  - 카드 본문 폰트 14→16, 매장명 18→22, 시각(우측) 18→26 (JetBrains Mono)
  - `--text-muted` 회색 → **검정(--text-primary)** 로 일괄 치환 (Dead-state 카드 원칙과 충돌 — 4번 참고)
  - hover 시 `background: var(--bg-hover)` + 텍스트는 그대로 검정 유지(현재는 hover 시 흐려지는 문제)
- **DESIGN.md 충돌 처리**
  - 기존 "Dead-state 카드" 원칙(온라인 정상 매장은 opacity 낮춰 배경으로 물러남)을 사용자가 명시적으로 뒤집고 있음 → Decisions Log에 "사용자 요청으로 정상 매장도 본문 가독성 우선" 명시
- **테스트**
  - Playwright/agent-browser: 24인치 모니터 + 1m 거리 가독성 스크린샷 비교 (before/after)
- **계약 변경**: 없음

### T2. 메인관제 카테고리 필터 (Frontend + Backend)

요구사항: "전체 / 출입문 / 냉장고 / 금고 …" 클릭형 필터. **카테고리별로 WS 알림 분기**.

- **데이터 모델 — 이미 존재**
  - `entity_types`: id, name(예: "냉장고"), state_mapping 등
  - `store_entities.type_id` → `entity_types.id`
  - `alert_events.type_id` → `entity_types.id` (이미 컬럼 존재)
- **결정사항 (A안 vs B안)**
  - **A안 — 서버사이드 필터 (권장)**: `GET /alerts?type_id=3&store_id=...` 쿼리파라미터 추가. WS는 `alert.new` 그대로 보내고 프론트에서 type_id로 분기. URL은 `/?cat=fridge` (서브경로 X) — Next.js App Router에서는 search param 사용. 메인페이지 1개 유지.
  - **B안 — 서브 페이지**: `/category/fridge`, `/category/door` 라우트 분리. 사용자 원안에 더 가까움. 카테고리 추가 시 라우트 자동 생성.
  - 권장: **A안** — entity_types는 동적이라 (관리자가 추가 가능) 정적 라우트 부적합. 탭/필터 칩 UI로 시각적으로는 동일한 UX 가능.
- **WS 메시지 변경**
  - `alert.new` 페이로드에 `type_id`, `type_name` 추가 (이미 있을 가능성 — 확인 필요)
  - 변경 시 `shared/contracts/api.md` WS 섹션 업데이트 + `to_frontend/`·`to_edge/` 통지
- **REST 변경**
  - `GET /alerts?type_id=...` 쿼리 지원 (없으면 전체)
  - `GET /entity_types`는 이미 존재 (확인 후 활용)
- **엣지케이스**
  - 미분류(`type_id IS NULL`) 알림 — "전체"에는 보이고, 개별 카테고리 어디에도 안 보임. 별도 "기타" 탭? → review에서 결정
  - 카테고리 탭 활성 상태에서 다른 카테고리 알림 도착 시 배지 카운트 표시 여부
- **테스트**
  - 백엔드: `GET /alerts?type_id=3` 필터 단위테스트
  - 프론트: WS 수신 시 active 카테고리 외 알림이 화면에 새로 안 뜨는지

### T3. 매장명 옆 중요도 표시 (Frontend only)

- 현재 `StoreGrid.tsx`에서 중요도는 카드 상단 우측 별점으로 표시 중. **알림 행(`AlertList.tsx`) 매장명 옆**에도 추가 요청.
- **결정사항**: 별점 5개 채움형(`★★★★☆`) vs 숫자 배지(`5★`) — 가독성 ↑ 위해 숫자 배지 권장 (T1과 일관성)
- **테스트**: 시각 회귀

### T4. 매장관리 상태 + 보관 워크플로우 (Frontend + Backend, 신규 모델)

요구사항이 두 부분 — 분리해서 봄.

#### T4a. 온·오프라인 시인성

- 현재 텍스트 라벨 + 색상 도트. 요구: **더 명확히** (색상변경 또는 직관적 표시).
- **결정**: 매장 카드 좌측에 두꺼운 4px 좌측 보더 (online=`--success`, offline=`--accent`) + 카드 배경 미세 톤 차이. heartbeat 90초 timeout으로 offline 판정되는 기존 로직 변경 X.
- 계약 변경 없음.

#### T4b. 보관(처리중) 워크플로우

요구사항: "보관(처리중) → 메모 작성 → 완료 → 보관, 보관로그"

해석: 알림에 대해 **즉시 확인 외에 "보관 처리"** 라는 중간 상태가 필요. 메모를 첨부할 수 있고, 나중에 완료 처리 가능. 보관/완료 이력은 보관로그로 조회.

- **신규 DB 테이블**: `alert_archives`
  ```
  id              bigserial PK
  alert_event_id  bigint FK alert_events(id) UNIQUE
  archived_by     bigint FK users(id)
  archived_at     timestamptz
  status          text  -- 'archived' | 'completed'
  memo            text
  completed_by    bigint FK users(id) NULL
  completed_at    timestamptz NULL
  completed_memo  text NULL
  ```
- **`alert_events` 변경**: 없음 (archive 여부는 join으로 판정) — 단, 인덱스 `alert_archives(status, archived_at desc)` 추가.
- **REST 신규**
  - `POST /alerts/{id}/archive`  body: `{ "memo": "..." }`
  - `POST /alerts/{id}/complete` body: `{ "memo": "..." }`
  - `GET /archives` query: `status`, `store_id`, `from`, `to`, `page` — 보관로그 조회
- **WS 신규**
  - `alert.archived` (id, archived_by, memo)
  - `alert.completed` (id, completed_by, memo)
- **UX**
  - VideoPopup 하단에 [확인 / 보관] 두 버튼 (현재 [✓ 확인] 하나)
  - 보관 누르면 메모 입력 모달 → 저장 후 알림 리스트에서 제거 (확인과 동일하게 사라짐)
  - 매장 상세에 "보관로그" 탭 신규
- **엣지케이스**
  - 동일 알림에 archive 후 다른 사람이 acknowledge — 충돌 처리: archive 상태에서는 acknowledge 차단 (또는 archive가 acknowledge를 implicit하게 포함)? → review 토픽
  - archive 후 같은 entity에서 동일 type 알림이 또 발생 — 새 alert로 처리 (기존 흐름과 동일)
  - 보관 후 메모 수정 — `PUT /archives/{id}` 필요한가?
- **테스트**
  - alembic 마이그레이션 down/up 검증
  - archive→complete 라이프사이클 통합테스트
  - 보관로그 페이지네이션

### T5. CTRL서버 양방향 연동 (Backend 중심)

요구사항: "CTRL서버와 연동 - 매장별 시간 get, 변경 시 iot 서버로 post"

- **현재 상태**: `external_poll.py`가 외부서버 → IoT(이 서버) 단방향 폴링만 구현. is_manual=0 행만 덮어씀. `POST /stores/{id}/schedules/sync` 트리거.
- **요구 해석**: CTRL서버에서 매장 운영시간을 변경하면 IoT 서버로 push (반대 방향). 또는 IoT에서 변경 시 CTRL로 push.
- **모호함 — review에서 명확화 필요**
  - 어느 쪽이 source of truth?
  - polling 주기 vs webhook?
- **결정 후보**
  - **C안 — CTRL→IoT webhook**: CTRL이 변경 시 `POST /api/v1/external/schedules` 호출 → 토큰 인증 → DB 반영
  - **D안 — IoT→CTRL push**: `PUT /stores/{id}/schedules/{day}` 성공 시 CTRL로 `POST {EXTERNAL_SERVER_URL}/api/schedules/{store_id}` 발사 (이미 토큰 있음)
  - **E안 — 양방향**: C+D 모두
- **엣지케이스**
  - 동시 변경 충돌 — last-write-wins vs version 필드
  - CTRL 다운 시 재시도 큐 — 현재 코드는 try/except로 삼킴, 재시도 없음. 정책 결정 필요.
- **테스트**
  - mock CTRL 서버로 E2E
- **계약 변경**: `shared/contracts/api.md`에 external 엔드포인트 섹션 신설

### T6. 일괄확인 (Backend + Frontend)

이안 사용자 요청. 사무실 실사용에서 알림이 다량 쌓일 때 일괄 처리 필요.

- **REST 신규**
  - `POST /alerts/acknowledge-all` body: `{ "store_id": 30584 (optional), "type_id": 3 (optional), "ids": [1,2,3] (optional) }`
  - 셋 중 우선순위: `ids > type_id+store_id > 전체`
  - 응답: `{ "acknowledged_count": N }`
- **WS**
  - 기존 `alert.acknowledged` 메시지를 ids 배열로 보낼 수 있게 확장 또는 신규 `alert.acknowledged_batch` 이벤트
  - **권장**: 신규 `alert.acknowledged_batch` (id list + acknowledged_by + acknowledged_at) — 기존 single ack 메시지와 분리해 프론트 처리 단순화
- **UX**
  - 알림 리스트 상단에 [일괄확인] 버튼
  - 현재 필터(카테고리/매장)가 적용되어 있으면 "필터된 알림만 일괄확인" 으로 동작 (사용자 의도와 부합)
  - 확인 다이얼로그 — "12개의 알림을 확인 처리합니다. 계속하시겠습니까?"
- **엣지케이스**
  - 일괄확인 도중 새 알림 도착 — 도착 시점 alert는 제외 (요청에 담긴 ids만 처리)
  - 중요도 ≥4 알림은 일괄확인에서 제외할지 옵션화? → 안전 측면에서 review 토픽
- **성능**
  - 한 번에 수백 건 처리 가능성 — 단일 UPDATE + WS 단일 메시지로 처리. N+1 금지.
- **테스트**
  - 100건 일괄확인 latency < 500ms
  - 동시 ack/배치 ack 충돌

---

## 2. 팀 분배 (오케스트레이터 안)

| Task | Backend | Frontend | Edge | 종속 |
|---|---|---|---|---|
| T1 시인성 |  | ● |  | — |
| T2 카테고리 | ● (alerts 필터) | ● (UI) |  | — |
| T3 중요도 |  | ● |  | T1 |
| T4a 온·오프라인 |  | ● |  | T1 |
| T4b 보관 워크플로우 | ● (모델·API·WS) | ● (UI·로그탭) |  | T2 |
| T5 CTRL 연동 | ● |  |  | — |
| T6 일괄확인 | ● (API·WS) | ● (버튼·다이얼로그) |  | T2 |

병렬 실행 가능 묶음:
- **레인 A (Frontend)**: T1 → T3 → T4a → T2-UI → T6-UI → T4b-UI
- **레인 B (Backend)**: T2-API → T4b-API → T6-API → T5
- **합류 지점**: T2(WS payload 합의), T4b(WS 이벤트 합의), T6(WS 이벤트 합의)

---

## 3. 계약 변경 영향 (shared/contracts/api.md)

신규/변경 엔드포인트:
- `GET /alerts` — `type_id`, `store_id` 쿼리 추가
- `POST /alerts/acknowledge-all`
- `POST /alerts/{id}/archive`
- `POST /alerts/{id}/complete`
- `GET /archives`
- `POST /api/v1/external/schedules` (C안 채택 시)

신규 WS 이벤트:
- `alert.acknowledged_batch`
- `alert.archived`
- `alert.completed`

`alert.new` payload 변경: `type_id`, `type_name` 포함 (이미 있을 수 있음 — 확인)

→ 계약 PR 한 번에 묶어서 backend·frontend 양쪽에 통지.

---

## 4. DB 마이그레이션

신규: `alert_archives` 테이블 + 인덱스
변경 없음: `alert_events`(기존 컬럼 재활용)

alembic revision 1개:
- `add_alert_archives_table`

---

## 4-1. plan-eng-review 락인 결정사항 (Phase 1)

- **Scope**: Phase 1만 (T1·T2·T3·T4a·T6). Phase 2는 T4b만, T5 CTRL 연동은 이번 스프린트 제외.
- **A1 (arch)**: T2 카테고리 식별자 `type_id`. `alert.new` WS 페이로드에 `type_id` 추가. `type_name`은 이미 포함.
- **A2 (arch)**: T6 일괄확인 body에 `ids` 또는 (`store_id` | `type_id`) 최소 하나 필수. 빈 body 400. 프론트는 현재 필터(카테고리·매장)를 자동 전달.
- **A3 (arch)**: T2 프론트는 search param 기반 탭/칩 (`/?cat=<type_id>`). 별도 라우트 X.
- **C1 (code)**: T1은 `globals.css` 토큰 갱신만 (`--text-muted` 짙은 톤, 신규 폰트 사이즈 토큰). DESIGN.md Decisions Log에 "Dead-state 카드 원칙 완화 — 사용자 요청으로 온라인도 본문 가독성 우선" 명시. 컴포넌트 인라인 색 변경 금지.
- **C2 (code)**: `app/services/alert_ack.py:ack_alerts(db, ids, user) -> [acknowledged_ids]` 신규 service. `WHERE acknowledged_by IS NULL` 가드 포함. 단일 라우터(`acknowledge_alert`) + 일괄 라우터(`acknowledge_alerts_bulk`) 둘 다 이 service 호출. WS broadcast도 공통화.
- **T (test)**: Backend `tests/test_alerts.py` 신규 (단일·일괄·필터·가드 커버). Frontend는 이번에 jest infra 도입 X — `/qa` 스킬 + agent-browser 수동 시나리오로 회귀 검증. F-TEST-INFRA는 별도 후속.

---

## 4-2. Test review — Phase 1 coverage 다이어그램

```
[+] backend/app/routers/alerts.py
  ├── list_alerts (변경: type_id 쿼리)
  │   ├── [GAP] type_id=None → 전체 미확인 반환
  │   ├── [GAP] type_id=N → 해당 카테고리만
  │   └── [GAP] type_id=999(존재하지 않음) → 빈 배열
  ├── acknowledge_alert (변경: service 위임 + IS NULL 가드)
  │   ├── [GAP] (REGRESSION) 이미 ack된 alert에 또 ack → 무시
  │   ├── [GAP] 존재하지 않는 alert_id → 404
  │   └── [GAP] WS alert.acknowledged 발사
  └── acknowledge_alerts_bulk (신규)
      ├── [GAP] body {ids:[1,2,3]} → 3건 ack, batch WS 1회
      ├── [GAP] body {store_id: X} → 해당 매장 미확인 ack
      ├── [GAP] body {type_id: Y} → 해당 type 미확인 ack
      ├── [GAP] body {store_id: X, type_id: Y} → AND 필터
      ├── [GAP] body {} (빈) → 400
      ├── [GAP] ids에 이미 ack 포함 → IS NULL 가드로 자동 제외
      └── [GAP] WS alert.acknowledged_batch 단일 메시지 (ids 배열)

[+] backend/app/services/alert_ack.py (신규)
  └── ack_alerts(db, ids, user) -> acknowledged_ids
      ├── [GAP] 동시 ack 시뮬레이션 (같은 alert 두 트랜잭션) → 한쪽만 성공
      └── [GAP] 빈 ids → 빈 반환

[+] backend/app/services/event_processor.py (변경: type_id WS payload)
  └── [GAP] 회귀 — alert.new payload에 type_id 포함 (type_name은 기존)

USER FLOWS (수동 QA / agent-browser 시나리오)
  ├── [→QA] 시인성: 메인 카드·모달의 폰트/색 변경 시각 검증 (DESIGN.md 토큰)
  ├── [→QA] 카테고리 필터: 칩 클릭 → URL ?cat=N → 해당 알림만 → 새 알림 도착 시 카테고리 일치 여부에 따른 표시
  ├── [→QA] 일괄확인: 100건 미확인 상태에서 일괄확인 → 확인 다이얼로그 → N건 사라짐 → 다른 클라(2탭)도 동기화
  ├── [→QA] (REGRESSION) 단일 ack: VideoPopup [✓확인] → 1건 사라짐 → 다른 클라 동기화
  ├── [→QA] 매장명 옆 중요도 별표(또는 숫자) 표시
  ├── [→QA] (REGRESSION) 매장 그리드 온/오프라인 — 보더 색 변경, 90초 timeout 동작 유지
  └── [→QA] (REGRESSION) DESIGN.md Decisions Log 갱신 후 다른 화면 위화감 없는지

COVERAGE: backend 13개 path 중 0건 자동 테스트 → 본 plan으로 13/13 추가
QUALITY: ★★★(behavior+edge+error) 목표 — 모든 GAP를 ★★★로 끌어올림
REGRESSION: 단일 ack + alert.new payload + 매장 온오프라인 — 명시 회귀 테스트 / 수동 시나리오 보유
```

### Test plan artifact (별도 파일은 안 만들고 본 plan에 통합)

- backend 신규: `backend/tests/test_alerts.py`
  - 시나리오: list / 단일 ack / 일괄 ack / IS NULL 가드 / batch WS broadcast / body validation
- backend 회귀: `backend/tests/test_websocket.py` 확장 — `alert.new` payload에 `type_id` 검증
- frontend 수동 QA: 위 [→QA] 시나리오 7개를 `/qa` 스킬로 1회 + agent-browser 스크린샷 before/after

---

## 4-3. Performance review

No issues, moving on.

근거:
- `GET /alerts` 는 미확인 알림만(`acknowledged_by IS NULL`) 조회 — 정상 운영 시 수십~수백 건. type_id 필터 추가는 인덱스 영향 미미.
- 일괄 ack: 단일 `UPDATE ... WHERE id IN (...) AND acknowledged_by IS NULL` — DB가 처리. 100건도 < 50ms.
- WS batch broadcast: 메시지 1개에 ids 배열 (≤1KB), 5 클라이언트 × 1 = 5번 send. 부담 없음.
- 카테고리 필터 클라이언트 사이드 `.filter()` — 수백 건 데이터셋에 무시 가능.

---

## 4-4. What already exists

이번 plan이 새로 만들 필요 없는 기반:

| 기반 | 위치 | 활용 |
|---|---|---|
| `entity_types` 테이블 (id, name) | `backend/app/models/entity_types.py` | T2 카테고리 식별자 (type_id) 그대로 사용 |
| `store_entities.type_id` FK | `backend/app/models/store_entities.py:17` | alert 생성 시 entity → type_id 추적 |
| `alert_events.type_id`, `type_name` | `backend/app/models/alert_events.py` | T2 필터 컬럼 이미 존재 |
| `alert.new` WS의 `type_name` | `backend/app/services/event_processor.py:170` | T2 라벨용 — 그대로 사용 |
| `acknowledge_alert` 라우터 | `backend/app/routers/alerts.py:27` | C2 service로 위임 형태로 리팩터 |
| `StoreGrid.tsx` 별표 패턴 | `frontend/components/StoreGrid.tsx:113-116` | T3에 동일 패턴 재사용 |
| `useWebSocket` 훅 | `frontend/hooks/useWebSocket.ts` | 신규 `alert.acknowledged_batch` 핸들러만 추가 |
| `DESIGN.md` 토큰 시스템 | `frontend/DESIGN.md`, `app/globals.css` | T1 토큰 갱신 — 토큰 자체는 이미 있음 |
| `monitoring_schedules` + `is_in_schedule` 로직 | `backend/app/services/event_processor.py`, `models/monitoring_schedules.py` | 변경 없이 그대로 |

---

## 4-5. NOT in scope (명시적 deferral)

| 항목 | 사유 |
|---|---|
| T5 CTRL서버 양방향 연동 | 사용자 명시 제외. 외부 시스템 정합성·source-of-truth 협의 선행 필요. 별도 office-hours/plan 라운드 후 진행. |
| T4b 보관(처리중) 워크플로우 + 메모 + 보관로그 | Phase 2. 신규 `alert_archives` 테이블 + UX 결정(archive와 ack 관계, 메모 수정 등) 미결. |
| Frontend Jest/RTL/Playwright infra 도입 | F-TEST-INFRA로 분리. Phase 1은 수동 QA로 진행. infra는 다음 스프린트에서 한 번에. |
| 카테고리별 미수신 알림 배지 카운트 | T2 부속 옵션. 이번엔 active 카테고리 표시만, 다른 카테고리 카운트는 별도. |
| 알림음 시스템 자체 개편 | T1은 시각 시인성만, 소리는 그대로. |
| 보관로그 검색·필터 고도화 | T4b 1차는 매장·기간 단순 필터까지. |
| entity_types CRUD 관리 UI 개편 | 이번 스프린트 외. |

---

## 4-6. Implementation Tasks (build-actionable)

이번 라운드 P1(=Phase 1 필수). 각 task는 backend / frontend 워크트리로 분배.

- [ ] **T1-FE (P1, human: ~3h / CC: ~30min)** — frontend — 토큰 갱신 + DESIGN.md Decisions Log
  - Files: `frontend/app/globals.css`, `frontend/DESIGN.md`, (변경 X) `components/StoreGrid.tsx`·`AlertList.tsx`·`VideoPopup.tsx`
  - Verify: agent-browser 스크린샷 before/after, 24인치 모니터 가독성
- [ ] **T2-BE (P1, human: ~1h / CC: ~10min)** — backend — `GET /alerts?type_id=N` + `alert.new` payload에 `type_id` 추가
  - Files: `backend/app/routers/alerts.py:list_alerts`, `backend/app/services/event_processor.py:170`, `backend/app/schemas/alerts.py`
  - Verify: `uv run pytest tests/test_alerts.py` (해당 테스트 신규 작성)
- [ ] **T2-FE (P1, human: ~4h / CC: ~30min)** — frontend — CategoryFilter 신규, search param 라우팅, AlertList 필터링
  - Files: `frontend/components/CategoryFilter.tsx`(신규), `frontend/app/page.tsx`, `frontend/components/AlertList.tsx`, `frontend/hooks/useWebSocket.ts`, `frontend/lib/types.ts`
  - Verify: agent-browser — 카테고리 칩 클릭, URL 변경, 알림 필터링
- [ ] **T3-FE (P1, human: ~30min / CC: ~5min)** — frontend — 알림 행 매장명 옆 중요도 표시
  - Files: `frontend/components/AlertList.tsx`
  - Verify: 시각 확인
- [ ] **T4a-FE (P1, human: ~1h / CC: ~10min)** — frontend — 매장 카드 온/오프라인 좌측 보더 강조
  - Files: `frontend/components/StoreGrid.tsx`, `frontend/app/globals.css`
  - Verify: heartbeat 90초 timeout 동작 유지 + 시각 확인
- [ ] **T6-BE (P1, human: ~3h / CC: ~25min)** — backend — `ack_alerts` service + `acknowledge_alerts_bulk` 라우터 + batch WS
  - Files: `backend/app/services/alert_ack.py`(신규), `backend/app/routers/alerts.py` (refactor + 신규 endpoint), `backend/app/websocket.py` (broadcast batch helper), `backend/tests/test_alerts.py`(신규)
  - Verify: `uv run pytest tests/test_alerts.py -v`
- [ ] **T6-FE (P1, human: ~2h / CC: ~20min)** — frontend — 일괄확인 버튼 + 다이얼로그 + WS batch 핸들러
  - Files: `frontend/components/AlertList.tsx`, `frontend/hooks/useWebSocket.ts`, `frontend/lib/api.ts`(필요 시), `frontend/lib/types.ts`
  - Verify: agent-browser — 일괄확인 다이얼로그 → N건 사라짐 → 2탭 동기화
- [ ] **CONTRACT (P1, human: ~30min / CC: ~10min)** — orchestrator — 계약 문서 갱신
  - Files: `shared/contracts/api.md` (`GET /alerts?type_id`, `POST /alerts/acknowledge-all`, WS `alert.acknowledged_batch`, `alert.new`에 `type_id`)
  - Verify: backend/frontend 양쪽 팀 inbox에 통지

### 의존성

```
CONTRACT  ─┬─▶ T2-BE ─┐
           ├─▶ T6-BE ─┤
           │           ├─▶ T2-FE
           │           ├─▶ T6-FE
           └─▶ T1-FE ─┴─▶ T3-FE ─▶ T4a-FE
```

CONTRACT 락인 후 BE·FE 두 레인이 진짜 병렬. UI 작업은 토큰 갱신(T1-FE) 이후 시각 작업이 차례.

---

## 4-7. Worktree parallelization

| Lane | 워크트리 | Task 순서 | 공유 모듈 |
|---|---|---|---|
| A (orchestrator) | `pcbang-sensor-project/` | CONTRACT → review·통지 | `shared/contracts/api.md` |
| B (backend) | `pcbang-sensor-backend/` | T2-BE → T6-BE | `backend/app/routers/alerts.py`, `services/`, `tests/test_alerts.py` |
| C (frontend) | `pcbang-sensor-frontend/` | T1-FE → T3-FE → T4a-FE → T2-FE → T6-FE | `frontend/app/`, `components/`, `hooks/`, `lib/`, `globals.css`, `DESIGN.md` |

A는 B·C 시작 직전 1회만 작업. B·C는 CONTRACT 후 완전 병렬.

**충돌 위험**:
- `shared/contracts/api.md` 와 `shared/{팀}/status.md` 는 오케스트레이터가 중앙화 → push race 차단.
- B·C는 다른 워크트리·다른 파일군 → 머지 충돌 없음.

---

## 4-8. Failure modes (운영 안전)

| 코드패스 | 한 가지 현실 실패 | 테스트 | 에러 핸들링 | 사용자에게 보이는 메시지 |
|---|---|---|---|---|
| `ack_alerts` 동시성 | 두 admin이 같은 alert에 동시 ack | `tests/test_alerts.py` IS NULL 가드 케이스 | UPDATE row-lock + IS NULL 자연 가드 | 한쪽 ack count = 0 — UI는 이미 사라진 상태로 일관됨 |
| `acknowledge_alerts_bulk` body validation | 빈 body 또는 ids = [] | `tests/test_alerts.py` 400 케이스 | 400 + `{code:"INVALID_BODY"}` | 다이얼로그가 "필터 조건 없음" 표시 |
| `alert.acknowledged_batch` WS broadcast | broadcast 도중 일부 클라 disconnect | (수동) 2탭에서 1탭 닫고 일괄확인 | manager.broadcast는 dead conn 무시 | 살아있는 클라만 동기화 |
| 카테고리 필터 — type_name 미사용 시 라벨 누락 | entity_types name이 빈 문자열 | (백엔드 시드) | UI에서 빈 라벨 시 "(미분류)" fallback | "(미분류)" 칩 |
| T1 토큰 변경 후 다른 화면 위화감 | 로그 페이지·매장 관리에서 회색 의도된 곳까지 검정으로 | (수동 QA 시나리오) | DESIGN.md Decisions Log에 영향 범위 명시 | 시각적 회귀 |

**Critical gap**: 없음. 모든 신규 path는 backend 자동 테스트 또는 수동 QA로 커버 예정.

---

## 5. 위험 / 미결 (review 후 해소됨)

검토 라운드 종료. 다음 항목은 모두 4-1 락인 또는 4-5 NOT in scope로 정리:

- ~~T2 A안/B안~~ → A3 락인 (search param 탭)
- ~~T4b archive vs ack 관계~~ → Phase 2 별도 라운드
- ~~T4b status enum~~ → Phase 2 별도 라운드
- ~~T5 source of truth~~ → 4-5 NOT in scope (이번 제외)
- ~~T6 중요도≥4 차단 옵션~~ → 차단 X, 단순 일괄. 운영 피드백으로 후속 결정
- ~~T4b broadcast~~ → Phase 2 별도 라운드
- ~~T1 vs Dead-state 원칙 충돌~~ → C1 락인 (사용자 가독성 우선)

## 6. 테스트 커버리지 목표 (4-2 결과 반영)

- Backend `tests/test_alerts.py` — 13개 시나리오 ★★★ 목표
- Backend `tests/test_websocket.py` — `alert.new` payload type_id 회귀 추가
- Frontend `/qa` 스킬 + agent-browser — 7개 시나리오 (시인성·카테고리·일괄확인·단일 ack 회귀·중요도·온오프라인·DESIGN.md 회귀)

## 7. 후속 TODO 제안 (이번 스프린트 외)

다음 plan-eng-review 라운드 또는 TODO 목록에 추가:

- **T5-EXT**: CTRL서버 양방향 연동 (사용자 명시 제외 — 외부 시스템 정합성 협의 후)
- **T4b-ARCHIVE**: 보관(처리중) 워크플로우 + 메모 + 완료 + 보관로그 (Phase 2 별도 라운드)
- **F-TEST-INFRA**: Frontend Jest + RTL + Playwright setup
- **T2-CATEGORY-BADGE**: 비활성 카테고리에 미수신 알림 카운트 배지
- **T1-SOUND**: 중요도 4·5 강조 알림음 재검토 (기존 A3 완료지만 시인성 변경과 함께 톤 점검)
- **CTRL-SERVER-CONFIG**: CTRL서버 연동을 위한 webhook secret·재시도 큐 설계

---

## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | `/plan-ceo-review` | Scope & strategy | 0 | — | not run |
| Codex Review | `/codex review` | Independent 2nd opinion | 0 | — | not run (skipped by user) |
| Eng Review | `/plan-eng-review` | Architecture & tests (required) | 1 | CLEAR (PLAN) | 6 issues, 0 critical gaps |
| Design Review | `/plan-design-review` | UI/UX gaps | 0 | — | not run |
| DX Review | `/plan-devex-review` | Developer experience gaps | 0 | — | n/a (no developer-facing surface) |

- **UNRESOLVED**: 0 (all decisions locked in §4-1 or deferred to §4-5/§7)
- **VERDICT**: ENG CLEARED — ready to implement Phase 1 (T1·T2·T3·T4a·T6). Phase 2(T4b) requires a separate /plan-eng-review round. T5 CTRL is excluded by user.


## 5. 위험 / 미결 (review에서 결정 필요)

1. **T2 — A안(쿼리) vs B안(라우트)**: 카테고리 동적 추가 가능성, URL 공유 가능성 trade-off
2. **T4b — archive와 acknowledge의 관계**: mutually exclusive? archive가 ack를 포함? 별도 상태머신?
3. **T4b — `alert_archives.status` enum**: archived / completed 만? 향후 cancelled / re-opened 필요?
4. **T5 — source of truth**: CTRL인가 IoT인가? 양방향이면 충돌 해소 정책
5. **T6 — 중요도 ≥4 일괄확인 차단 옵션**: 안전 대 편의 trade-off
6. **T4b WS broadcast**: archive 정보가 다른 5명 관제자에게 실시간 동기화되어야 하는가, 본인만 보는가?
7. **시인성(T1)이 DESIGN.md "Dead-state 카드" 원칙과 충돌** — 어느 쪽을 살릴지 명시 결정

---

## 6. 테스트 커버리지 목표

- Backend: 신규 라우터 happy-path + 권한(401) + 충돌(409) 케이스
- Frontend: WS reducer 단위 테스트 (카테고리 분기, 일괄확인 적용), Playwright로 archive 라이프사이클 1개
- 통합: docker-compose.test로 full stack alert→archive→complete 1개 시나리오

---

## 7. 비전제 (Non-goals)

- entity_types CRUD UI 개편 (이번 스프린트 외)
- 알림음 시스템 자체 개편 (T1 시각만)
- 보관로그 검색/필터 고도화 (1차는 매장·기간 단순 필터만)
- CTRL 서버 양방향 동시화 충돌 해소 UI (1차는 last-write-wins 가정)
