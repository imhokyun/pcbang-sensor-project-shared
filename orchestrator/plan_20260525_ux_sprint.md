# 2026-05-26 UX 스프린트 — 실행 계획 초안

원본 요구사항: [`shared/20260526_dev.md`](../20260526_dev.md)
검토 단계: **`/plan-eng-review` 진행 중 (Scope: Phase 1 락인 대상, Phase 2는 후속 스프린트)**
오케스트레이터: orchestrator (main 워크트리)

## Phase 분할 (Step 0 — Scope Challenge 결과)

복잡도 게이트 트리거 (20+ files, 5+ 신규 모델·라우터·컴포넌트) → **두 Phase로 분할**.

| Phase | Task | 비고 |
|---|---|---|
| **Phase 1 — 즉시 UX 효과** | T1 시인성, T2 카테고리 필터, T3 중요도, T4a 온·오프라인, T6 일괄확인 | 이번 스프린트. DB 모델 변경 없음. WS 페이로드 확장만. |
| **Phase 2 — 모델·외부연동** | T4b 보관 워크플로우, T5 CTRL 양방향 | 다음 스프린트. 신규 `alert_archives` 테이블, 외부 webhook. Phase 1 운영 피드백 후 진행. |

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

## 4-1. plan-eng-review 락인 결정사항 (Phase 1 한정)

진행 중인 review에서 확정된 항목 — 본 review가 완료되면 정식 review report로 대체된다.

- **Scope**: Phase 1만 락인 (T1·T2·T3·T4a·T6). Phase 2(T4b·T5)는 별도 라운드.
- **Architecture A1**: T2 카테고리 필터 식별자는 `type_id`. `alert.new` WS 페이로드에 `type_id` 추가 (`type_name`은 이미 포함).
- **Architecture A2**: T6 일괄확인은 body에 `ids` 또는 (`store_id` | `type_id`) 최소 하나 필수. 빈 body 400. 프론트는 현재 필터 상태(카테고리/매장)를 자동 전달.
- **Architecture A3**: T2 프론트는 search param 기반 탭/칩 (`/?cat=fridge`). 별도 라우트 X.
- **Code Quality C1**: T1은 `globals.css` 토큰 갱신 (`--text-muted`→짙은 톤, 신규 폰트 사이즈 토큰). DESIGN.md Decisions Log에 "Dead-state 카드 원칙 완화 — 사용자 요청으로 온라인도 본문 가독성 우선" 명시. 컴포넌트 인라인 수정 금지.

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
