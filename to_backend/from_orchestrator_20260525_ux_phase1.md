# Backend → Phase 1 UX 스프린트 작업 지시

발신: orchestrator (main)
일시: 2026-05-25
원본 plan: `shared/orchestrator/plan_20260525_ux_sprint.md`
원본 요구사항: `shared/20260526_dev.md`

---

## 작업 방식 — 반드시 TDD

각 task는 다음 순서로:

1. **테스트 먼저** — `backend/tests/test_alerts.py` 에 실패하는 시나리오 작성
2. **`uv run pytest tests/test_alerts.py -v` 로 실패 확인** (red)
3. **구현 작성** (green)
4. **다시 pytest로 통과 확인**
5. **리팩터 (필요 시)** — 테스트 그대로 두고 코드 정리

테스트 없이 구현부터 하지 말 것. 같은 turn 안에서 빨강→초록 사이클을 보여줘.

---

## Task 목록 (락인 결정 따름 — `plan_20260525_ux_sprint.md` §4-1)

### 1. CONTRACT 반영 확인 (READ ONLY)

`shared/contracts/api.md` 이미 갱신 완료. Phase 1 변경 4지점:

- `GET /alerts?type_id=N` 쿼리 추가
- `POST /alerts/acknowledge-all` 신규
- `alert.new` WS payload에 `type_id` 추가
- `alert.acknowledged_batch` WS 신규

### 2. T2-BE — `GET /alerts` type_id 필터 + `alert.new` type_id 추가

**테스트 추가** (`backend/tests/test_alerts.py` 신규):
- `test_list_alerts_no_filter` — 전체 미확인 알림 반환
- `test_list_alerts_with_type_id` — `?type_id=3` → 해당 카테고리만
- `test_list_alerts_unknown_type_id` — 존재하지 않는 type_id → 빈 배열

**구현**:
- `backend/app/routers/alerts.py:list_alerts` — `type_id: int | None = None` Query 파라미터 추가, where 절 조건부 추가
- `backend/app/services/event_processor.py:170 근처` — `alert.new` payload dict에 `"type_id": entity.type_id` 추가
- `backend/app/schemas/alerts.py:AlertOut` — `type_id: int | None` 필드 추가

**WS 회귀 테스트** (`backend/tests/test_websocket.py` 확장):
- `test_alert_new_payload_has_type_id` — broadcast된 alert.new 메시지에 type_id 필드 존재

### 3. T6-BE — `ack_alerts` service + `acknowledge_alerts_bulk` 라우터

**테스트 먼저** (`backend/tests/test_alerts.py`):
- `test_ack_alerts_service_basic` — service가 ids 받아 ack 처리 후 ids 반환
- `test_ack_alerts_service_is_null_guard` — 이미 ack된 alert 포함 시 자연 제외 (반환 ids에 없음)
- `test_ack_alerts_service_empty_ids` — 빈 배열 → 빈 배열 반환
- `test_acknowledge_single_uses_service` — 단일 라우터도 service 위임 (리팩터 회귀)
- `test_acknowledge_single_already_acked` — (REGRESSION) 이미 ack된 alert 두번째 호출 — count = 0, 200 OK
- `test_acknowledge_bulk_by_ids` — body `{ids: [1,2,3]}` → 3건 처리
- `test_acknowledge_bulk_by_store_id` — body `{store_id: X}` → 해당 매장 미확인 ack
- `test_acknowledge_bulk_by_type_id` — body `{type_id: Y}` → 해당 카테고리 ack
- `test_acknowledge_bulk_and_filter` — body `{store_id: X, type_id: Y}` → AND
- `test_acknowledge_bulk_empty_body_400` — 빈 body → 400 INVALID_BODY
- `test_acknowledge_bulk_broadcasts_batch_event` — WS `alert.acknowledged_batch` 단일 메시지 (ids 배열)

**구현 순서** (red→green→refactor):

1. `backend/app/services/alert_ack.py` 신규
   ```python
   async def ack_alerts(
       db: AsyncSession,
       *,
       ids: list[int] | None = None,
       store_id: int | None = None,
       type_id: int | None = None,
       user: User,
   ) -> list[int]:
       # WHERE: acknowledged_by IS NULL AND (ids 또는 store_id/type_id 필터)
       # UPDATE 한 번 → returning id → ack된 ids 반환
   ```
2. `backend/app/routers/alerts.py:acknowledge_alert` — service 호출로 위임
3. `backend/app/routers/alerts.py:acknowledge_alerts_bulk` 신규
   - body validation: `ids` 또는 (`store_id` | `type_id`) 최소 하나, 모두 비면 400
   - service 호출 후 결과 ids 받아 batch WS 발사
4. `backend/app/websocket.py` — `broadcast_acknowledged_batch(ids, user, now)` helper

---

## 진행 절차 (TDD)

```
cd /home/multion/production/pcbang-sensor-backend
cd shared && git pull origin main && cd ..   # 1) 계약 최신화
# 2) shared/backend/status.md, shared/orchestrator/plan_*.md 락인 결정 재확인
# 3) 테스트 신규 작성 → uv run pytest tests/test_alerts.py -v (RED 확인)
# 4) 구현
# 5) uv run pytest tests/test_alerts.py -v (GREEN 확인)
# 6) 필요 시 리팩터, 다시 pytest
# 7) git commit (작은 단위, 메시지에 [T2-BE], [T6-BE] 태그)
# 8) shared/backend/status.md 갱신
# 9) shared push 하지 말 것 — 오케스트레이터가 중앙화함
# 10) SendMessage로 team-lead에게 진행 보고
```

---

## 경계

- 워크트리 `/home/multion/production/pcbang-sensor-backend` 밖 파일 절대 수정 금지
- `shared/` 푸시 금지 — 읽기만. shared 변경은 모두 team-lead가 중앙화
- frontend 파일 절대 수정 금지
- `alert_archives` 테이블·CTRL 연동은 이번 스프린트 제외 — 손대지 말 것
