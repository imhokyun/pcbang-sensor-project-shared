# Backend → CTRL 동기화 작업 지시 (Phase 2)

발신: orchestrator (main)
일시: 2026-05-26
원본 plan: `shared/orchestrator/design_20260526_ctrl_sync.md`
원본 명세: `shared/20260526_ctrl_server_integration.md`
계약 갱신: `shared/contracts/api.md` §CTRL 동기화

---

## 작업 방식 — 반드시 TDD

각 task:

1. **테스트 먼저** — 시나리오 작성
2. **`uv run pytest tests/test_ctrl_sync.py -v`** → 빨강 확인
3. **구현**
4. **다시 pytest** → 초록 확인
5. **리팩터** (필요 시)

테스트 없이 구현부터 가지 마. 같은 turn에서 빨강 → 초록 사이클을 보여줘.

---

## Task 목록 (락인 결정 따름 — `design_20260526_ctrl_sync.md` §plan-eng-review 락인 결정사항)

### T2 BE-CTRL-SVC — services/ctrl_sync.py 신규

**테스트 먼저** (`backend/tests/test_ctrl_sync.py`, 신규):

`class TestDoWMapping`:
- `test_daysn_1_sunday` — daySn=1 → 6
- `test_daysn_2_monday` — daySn=2 → 0
- `test_daysn_3_tuesday` — daySn=3 → 1
- `test_daysn_4_wednesday` — daySn=4 → 2
- `test_daysn_5_thursday` — daySn=5 → 3
- `test_daysn_6_friday` — daySn=6 → 4
- `test_daysn_7_saturday` — daySn=7 → 5

`class TestFetchStoreDetail`:
- `test_fetch_success` — CTRL 200 → payload dict 반환 (mock httpx)
- `test_fetch_ctrl_401` — CTRL 401 → CtrlAuthError raise
- `test_fetch_timeout` — httpx TimeoutException → CtrlTimeoutError raise

`class TestApplyControlTimes`:
- `test_apply_updates_is_manual_zero_rows` — 7행 갱신, updated_count=7
- `test_apply_preserves_is_manual_one` (REGRESSION) — is_manual=1 한 행 → skipped_manual=1, 값 보존
- `test_apply_store_not_found` — stores row 없음 → updated_count=0, log 발생
- `test_apply_empty_begin_time_marks_inactive` — beginTime="" → is_active=0

`class TestSyncStoreSchedules`:
- `test_sync_happy_path` — fetch + apply 통합

`class TestHandleWebhookEvent`:
- `test_handle_created` — eventType=created → apply 호출
- `test_handle_updated` — eventType=updated → apply 호출
- `test_handle_time_updated` — eventType=time_updated → apply 호출
- `test_handle_deleted` — eventType=deleted → log only, DB 변경 X
- `test_handle_store_not_in_our_db` — storeNo 미존재 → 200 OK, log만
- `test_handle_idempotent` — 같은 payload 2회 → 동일 결과

→ 첫 `pytest` 실행 시 RED 확인 (구현 없음).

**구현**:

`backend/app/services/ctrl_sync.py` 신규:

```python
"""CTRL 시스템 동기화 서비스.

- _ctrl_daysn_to_our_dow: CTRL daySn → 우리 day_of_week 매핑
- fetch_store_detail: CTRL /api/store/store_detail 호출
- apply_control_times: controlTimes → monitoring_schedules 반영
- sync_store_schedules: fetch + apply 통합 (수동 버튼/sync 라우터)
- handle_webhook_event: webhook 진입점, eventType 분기
"""
from datetime import datetime, timezone
import httpx
import logging
from typing import Any

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.models.monitoring_schedules import MonitoringSchedule
from app.models.stores import Store

logger = logging.getLogger(__name__)


class CtrlError(Exception): pass
class CtrlAuthError(CtrlError): pass
class CtrlTimeoutError(CtrlError): pass
class CtrlStoreNotFoundError(CtrlError): pass


def _ctrl_daysn_to_our_dow(daysn: int) -> int:
    """CTRL daySn (1=일, 2=월, ..., 7=토) → 우리 day_of_week (0=월, ..., 6=일).

    수식: (daysn + 5) % 7
    검증: 1→6, 2→0, 3→1, 4→2, 5→3, 6→4, 7→5
    """
    return (daysn + 5) % 7


async def fetch_store_detail(store_no: int) -> dict[str, Any]:
    """CTRL 매장 상세 조회."""
    url = f"{settings.CTRL_API_BASE}/api/store/store_detail"
    headers = {"X-SERVER-API-KEY": settings.CTRL_API_KEY}
    try:
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.get(url, headers=headers, params={"storeNo": store_no})
        if resp.status_code == 401:
            raise CtrlAuthError("CTRL X-SERVER-API-KEY 무효")
        if resp.status_code == 400:
            raise CtrlStoreNotFoundError(f"CTRL store_no={store_no} 미존재")
        resp.raise_for_status()
        return resp.json().get("data", {})
    except httpx.TimeoutException as e:
        raise CtrlTimeoutError("CTRL 응답 timeout") from e


async def apply_control_times(
    db: AsyncSession, store_id: int, control_times: list[dict]
) -> dict:
    """controlTimes → monitoring_schedules 반영.

    Returns: {"updated_count": N, "skipped_manual": M}
    """
    # stores row 존재 확인
    store_exists = (await db.execute(
        select(Store.store_id).where(Store.store_id == store_id)
    )).scalar_one_or_none()
    if store_exists is None:
        logger.warning("ctrl_sync: store_not_found store_id=%s", store_id)
        return {"updated_count": 0, "skipped_manual": 0}

    now = datetime.now(timezone.utc)
    updated = 0
    skipped = 0

    for item in control_times:
        daysn = item.get("daySn")
        if daysn is None:
            continue
        dow = _ctrl_daysn_to_our_dow(daysn)
        begin = item.get("beginTime", "")
        end = item.get("endTime", "")
        is_active = 1 if (begin and end) else 0

        # is_manual=1 행은 보존
        manual_check = (await db.execute(
            select(MonitoringSchedule.is_manual).where(
                MonitoringSchedule.store_id == store_id,
                MonitoringSchedule.day_of_week == dow,
            )
        )).scalar_one_or_none()

        if manual_check == 1:
            skipped += 1
            continue

        # UPSERT
        existing = (await db.execute(
            select(MonitoringSchedule).where(
                MonitoringSchedule.store_id == store_id,
                MonitoringSchedule.day_of_week == dow,
            )
        )).scalar_one_or_none()

        if existing:
            existing.start_time = begin
            existing.end_time = end
            existing.is_active = is_active
            existing.updated_at = now
        else:
            db.add(MonitoringSchedule(
                store_id=store_id,
                day_of_week=dow,
                start_time=begin,
                end_time=end,
                is_active=is_active,
                is_manual=0,
                updated_at=now,
            ))
        updated += 1

    await db.commit()
    return {"updated_count": updated, "skipped_manual": skipped}


async def sync_store_schedules(store_id: int, db: AsyncSession) -> dict:
    """수동 동기화 (`POST /stores/{id}/schedules/sync` 진입점)."""
    detail = await fetch_store_detail(store_id)
    control_times = detail.get("controlTimes", [])
    return await apply_control_times(db, store_id, control_times)


async def handle_webhook_event(payload: dict, db: AsyncSession) -> dict:
    """Webhook 진입점 — eventType 분기."""
    event_type = payload.get("eventType")
    store_no = payload.get("storeNo")
    if event_type is None or store_no is None:
        return {"applied": "none", "reason": "missing_fields"}

    if event_type == "deleted":
        logger.info("ctrl_webhook: deleted store_no=%s (no DB change)", store_no)
        return {"store_id": store_no, "event_type": "deleted", "applied": "none",
                "updated_count": 0, "skipped_manual": 0}

    if event_type not in ("created", "updated", "time_updated"):
        logger.warning("ctrl_webhook: unknown event_type=%s", event_type)
        return {"applied": "none", "reason": "unknown_event_type"}

    control_times = payload.get("controlTimes", [])
    result = await apply_control_times(db, store_no, control_times)
    return {
        "store_id": store_no,
        "event_type": event_type,
        "applied": "schedules" if result["updated_count"] > 0 else "none",
        **result,
    }
```

`pytest -v` → GREEN 확인.

**커밋**: `[T2-BE-CTRL] services/ctrl_sync.py 신규 + TDD`

---

### T3 BE-WEBHOOK — routers/webhooks.py 신규

**테스트 먼저** (`backend/tests/test_ctrl_sync.py` 확장 또는 `test_webhooks.py` 신규):

- `test_webhook_secret_match_200` — `X-WEBHOOK-SECRET` 일치 + 정상 payload → 200
- `test_webhook_secret_mismatch_401` — 불일치 → 401
- `test_webhook_secret_missing_401` — 헤더 누락 → 401
- `test_webhook_invalid_body_422` — JSON 파싱 실패 → 422
- `test_webhook_missing_required_field_422` — storeNo 누락 → 422

**구현**:

`backend/app/routers/webhooks.py` 신규:

```python
from fastapi import APIRouter, Depends, Header, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import get_db
from app.services.ctrl_sync import handle_webhook_event

router = APIRouter(prefix="/webhooks", tags=["webhooks"])


async def _verify_webhook_secret(
    x_webhook_secret: str | None = Header(default=None, alias="X-WEBHOOK-SECRET"),
):
    if not x_webhook_secret or x_webhook_secret != settings.CTRL_WEBHOOK_SECRET:
        raise HTTPException(status_code=401, detail={"code": "INVALID_SECRET"})


@router.post("/ctrl/store", dependencies=[Depends(_verify_webhook_secret)])
async def receive_ctrl_store(payload: dict, db: AsyncSession = Depends(get_db)):
    # Pydantic 안 쓰고 dict로 받아 우리가 필요한 필드만 검증
    if "storeNo" not in payload or "eventType" not in payload or "controlTimes" not in payload:
        raise HTTPException(status_code=422, detail={"code": "MISSING_REQUIRED_FIELDS"})
    result = await handle_webhook_event(payload, db)
    return {"success": True, "data": result}
```

**커밋**: `[T3-BE-WEBHOOK] routers/webhooks.py + 인증 + TDD`

---

### T4 BE-SCHEDULES — schedules/sync 라우터 위임

**테스트** (`backend/tests/test_schedules.py` 확장 또는 신규):
- `test_sync_calls_ctrl_sync` — sync 라우터가 ctrl_sync.sync_store_schedules 호출
- `test_sync_returns_updated_skipped` — 응답 형식 `{success, data:{updated_count, skipped_manual}}`
- `test_sync_ctrl_401_returns_502` — CtrlAuthError → 502 CTRL_AUTH_FAILED
- `test_sync_ctrl_timeout_returns_504` — CtrlTimeoutError → 504 CTRL_TIMEOUT

**구현**: `backend/app/routers/schedules.py` 의 `sync_schedules` 함수 변경

```python
# 기존: from app.services.external_poll import poll_store_schedules
# 변경: from app.services.ctrl_sync import sync_store_schedules, CtrlAuthError, CtrlTimeoutError, CtrlStoreNotFoundError

@router.post("/stores/{store_id}/schedules/sync")
async def sync_schedules(
    store_id: int,
    db: AsyncSession = Depends(get_db),
    _user: User = Depends(get_current_user),
):
    try:
        result = await sync_store_schedules(store_id, db)
    except CtrlAuthError:
        raise HTTPException(502, detail={"code": "CTRL_AUTH_FAILED"})
    except CtrlTimeoutError:
        raise HTTPException(504, detail={"code": "CTRL_TIMEOUT"})
    except CtrlStoreNotFoundError:
        raise HTTPException(502, detail={"code": "CTRL_STORE_NOT_FOUND"})
    return {"success": True, "data": result}
```

`backend/app/services/external_poll.py` 상단에 DEPRECATED 주석 추가:
```python
"""DEPRECATED — services/ctrl_sync.py 가 대체.

이 모듈은 가상의 외부 polling endpoint를 가정한 prototype 코드.
실제 CTRL 시스템 연동은 ctrl_sync 로 이관됨. 향후 다른 외부 시스템 연동 시
참조용으로 코드는 보존하되 새 호출자 추가 금지.
"""
```

**커밋**: `[T4-BE-SCHEDULES] sync 라우터 ctrl_sync 위임 + external_poll deprecate`

---

### T5 BE-CONFIG — config.py + main.py + .env.example

**변경**:

`backend/app/config.py`:
```python
class Settings:
    ...
    CTRL_API_BASE: str = os.getenv("CTRL_API_BASE", "")
    CTRL_API_KEY: str = os.getenv("CTRL_API_KEY", "")
    CTRL_WEBHOOK_SECRET: str = os.getenv("CTRL_WEBHOOK_SECRET", "")
```

`backend/app/main.py`:
```python
from app.routers import auth, cameras, config, edge, entities, alerts, entity_types, go2rtc, logs, relays, schedules, stores, webhooks
...
app.include_router(webhooks.router, prefix="/api/v1")
```

`backend/.env.example`:
```ini
# ---------------------------------
# CTRL 시스템 연동
# ---------------------------------
CTRL_API_BASE=https://ctrl.example.com
CTRL_API_KEY=byIL129ZEUb9ySOy6iZPcr0lPCpS1ug3wi72QW4CWCBRfQt6mqRXSrV9HGXtZJHK
CTRL_WEBHOOK_SECRET=  # openssl rand -hex 32
```

**커밋**: `[T5-BE-CONFIG] CTRL env keys + webhook router 등록`

---

## 진행 절차 (TDD)

```bash
cd /home/multion/production/pcbang-sensor-backend
cd shared && git pull origin main && cd ..
# shared/orchestrator/design_20260526_ctrl_sync.md, shared/contracts/api.md 재확인

# T2 사이클
# 1. tests/test_ctrl_sync.py 신규 — DoW 매핑 시나리오 7개 + ApplyControlTimes 4개
# 2. uv run pytest tests/test_ctrl_sync.py -v  → RED
# 3. services/ctrl_sync.py 신규 구현
# 4. uv run pytest tests/test_ctrl_sync.py -v  → GREEN
# 5. SendMessage to team-lead 보고 + git commit [T2-BE-CTRL]

# T3, T4, T5 동일 패턴

# 작업 끝나면 idle 상태로 대기
```

## 경계

- 워크트리 `/home/multion/production/pcbang-sensor-backend` 안에서만
- frontend / edge / project root 파일 절대 수정 X
- shared push 절대 금지 — 읽기만, 변경은 team-lead 중앙화
- alembic 마이그레이션 추가 X (DB 스키마 변경 0이 락인)
- 비밀번호 5필드 복호화 코드 작성 X (A안 — 우리 DB에 안 저장)
- `system_config` 테이블 수정 X
