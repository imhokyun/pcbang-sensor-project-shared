# Frontend → Phase 1 UX 스프린트 작업 지시

발신: orchestrator (main)
일시: 2026-05-25
원본 plan: `shared/orchestrator/plan_20260525_ux_sprint.md`
원본 요구사항: `shared/20260526_dev.md`

---

## 작업 방식 — 행동 우선 (behavior-first), TDD 정신

이번 Phase 1에서는 jest infra 도입 안 함 (NOT in scope). 그러나 TDD 정신은 유지:

1. **시나리오 먼저 적기** — 각 task에 대해 agent-browser로 검증할 "성공 시나리오"를 텍스트로 명시 (commit 메시지 또는 PR body)
2. **agent-browser로 실패 확인** — 변경 전 상태 스크린샷, 시나리오 통과 못함 확인
3. **구현**
4. **agent-browser로 통과 확인** — 변경 후 상태 스크린샷, 시나리오 통과
5. **시각 회귀** — 시나리오에 없는 화면 (로그·매장관리 등)도 1회 둘러보기

테스트 인프라 없다고 그냥 "되겠지"로 가지 말 것. 시나리오 → 검증 → 구현 → 재검증 사이클은 동일.

---

## Task 목록 (락인 결정 따름 — `plan_20260525_ux_sprint.md` §4-1)

### 1. T1-FE — 시인성 (DESIGN.md 토큰 갱신, 컴포넌트 인라인 X)

**시나리오**:
- 메인 관제 카드의 매장명·시각·본문이 1m 거리 24인치에서 명확히 읽힘
- 회색이던 본문이 검정으로 표시되어 가독성 ↑
- hover 시 텍스트는 그대로 검정, 배경만 변화

**구현 (DESIGN.md Decisions Log 먼저 명시한 다음)**:
1. `frontend/DESIGN.md` Decisions Log 추가: "사용자 요청으로 Dead-state 카드 원칙 완화 — 온라인 카드도 본문 가독성 우선. `--text-muted`는 라벨/메타 정보 한정, 데이터 본문은 `--text-primary`."
2. `frontend/app/globals.css`:
   - `--text-muted` 톤 짙게 (기존 `#868e96` → 검정에 가까운 `#212529` 또는 적정 톤)
   - 신규 토큰: `--font-card-body` (16px), `--font-store-name` (22px), `--font-time` (26px JetBrains Mono)
3. 기존 컴포넌트는 토큰 참조만 — 인라인 색·폰트 사이즈 변경 금지

**검증**: agent-browser before/after 스크린샷 비교 + hover 인터랙션

### 2. T2-FE — 카테고리 필터 (search param 탭/칩)

**시나리오**:
- 메인 페이지에 entity_types 기반 칩 표시 (전체, 출입문, 냉장고, 금고...)
- 칩 클릭 → URL이 `/?cat=<type_id>` 로 변경 → 해당 카테고리 알림만 표시
- 새 알림 도착 시: 현재 active 카테고리와 일치하면 표시, 불일치하면 표시 안 함
- 전체 칩 클릭 → URL `/` → 모든 알림 표시

**구현**:
1. `frontend/lib/types.ts` — `Alert` 타입에 `type_id: number | null` 추가, `EntityType` 타입 신규(`{id, name}`)
2. `frontend/components/CategoryFilter.tsx` 신규 — entity_types 목록 (`GET /entity-types` 또는 init WS 메시지에서) + active tab + 클릭 시 `router.replace('/?cat=N')`
3. `frontend/app/page.tsx` — `useSearchParams()` 로 `cat` 읽고 AlertList에 `activeTypeId` props 전달
4. `frontend/components/AlertList.tsx` — `activeTypeId` 있으면 `.filter(a => a.type_id === activeTypeId)` 적용
5. `frontend/hooks/useWebSocket.ts` — `alert.new` 핸들러는 그대로 (모든 알림 수신, 필터링은 UI 레이어)

**검증**: agent-browser — 칩 클릭, URL 변경, 알림 표시 변화

### 3. T3-FE — 알림 행 매장명 옆 중요도 표시

**시나리오**:
- `AlertList.tsx`의 각 행에서 매장명 옆에 별점 또는 숫자로 중요도 (1~5) 표시

**구현**: `frontend/components/AlertList.tsx` — `StoreGrid.tsx:113-116`의 별점 패턴 그대로 재사용 (`{"★".repeat(alert.importance)}`)

### 4. T4a-FE — 매장 카드 온/오프라인 시인성

**시나리오**:
- 매장 카드 좌측에 4px 보더: online=`--success`, offline=`--accent`
- heartbeat 90초 timeout 동작은 기존 그대로 (수정 X)

**구현**: `frontend/components/StoreGrid.tsx` — 카드 컨테이너에 `borderLeft: '4px solid var(--success/--accent)'`. `frontend/app/globals.css` 에 필요 시 보조 토큰.

### 5. T6-FE — 일괄확인 버튼 + 다이얼로그 + WS batch 핸들러

**시나리오**:
- 알림 리스트 상단 [일괄확인] 버튼
- 현재 필터(카테고리/매장)가 적용된 상태면 그 범위만 일괄확인
- 클릭 → 확인 다이얼로그 "N개의 알림을 확인 처리합니다. 계속하시겠습니까?"
- 확인 → `POST /alerts/acknowledge-all` 호출 (body에 ids 또는 type_id)
- WS `alert.acknowledged_batch` 수신 → 해당 ids 한 번에 리스트에서 제거
- 다른 브라우저 탭에서도 동기화됨

**구현**:
1. `frontend/components/AlertList.tsx` — 상단 [일괄확인] 버튼 + ConfirmDialog
2. body 결정 로직: 현재 보이는(필터 적용된) alert ids를 명시 전달 (`{ids: [...]}`) — 가장 deterministic
3. `frontend/lib/api.ts` — `acknowledgeAlertsBulk(body)` 함수
4. `frontend/hooks/useWebSocket.ts` — `alert.acknowledged_batch` 핸들러: `ids[]`로 reducer에서 일괄 제거

**검증**: agent-browser — 2탭 동시 열기, 1탭에서 일괄확인 → 다른 탭 동기화 확인

---

## 진행 절차 (behavior-first)

```
cd /home/multion/production/pcbang-sensor-frontend
cd shared && git pull origin main && cd ..   # 1) 계약 최신화
# 2) shared/frontend/status.md, shared/orchestrator/plan_*.md 락인 결정 재확인
# 3) 시나리오 텍스트로 commit message draft (변경 전 agent-browser 스샷 첨부)
# 4) 구현
# 5) agent-browser로 시나리오 검증
# 6) git commit (작은 단위, [T1-FE], [T2-FE]... 태그)
# 7) shared/frontend/status.md 갱신
# 8) shared push 하지 말 것 — 오케스트레이터가 중앙화
# 9) SendMessage로 team-lead에게 진행 보고 + 스크린샷 파일 경로 명시
```

---

## 경계

- 워크트리 `/home/multion/production/pcbang-sensor-frontend` 밖 파일 절대 수정 금지
- `shared/` 푸시 금지 — 읽기만. shared 변경은 모두 team-lead가 중앙화
- backend 파일 절대 수정 금지
- `T4b 보관 워크플로우`·`T5 CTRL 연동`은 이번 스프린트 제외 — 손대지 말 것
- `jest.config.*`·`vitest.config.*`·새 `*.test.*` 파일 만들지 말 것 — Frontend 자동 테스트 infra는 별도 후속(F-TEST-INFRA)
