# Backend 작업 지시 (Rev.2)

**발신**: Orchestrator | **날짜**: 2026-04-08 | **브랜치**: dev/backend  
**전체 그림**: `shared/new_request_20260408_rev2.md` 먼저 읽어볼 것

> **Rev.1 대비 변경**: A4 방식 변경 — DB 폴링 제거, MQTT 수신 방식으로 전환

---

## 담당 작업 요약

| ID | 작업 | 의존성 |
|----|------|--------|
| A4-BE | 장시간 개방 — MQTT 구독 + AlertEvent 생성 | 없음 (Edge와 병렬 가능) |
| B1-BE | 매장 API 검색·페이지네이션 쿼리 파라미터 추가 | 없음 |

두 작업 모두 독립적으로 **병렬 처리** 가능합니다.

---

## A4-BE — 장시간 개방 감지 (Edge 감지 방식)

### 설계 요약

```
Edge가 asyncio 타이머로 감지 → MQTT publish
          │
          ▼  pcbang/{store_id}/entities/{ha_entity_id}/long_open
Backend MQTT 구독
          │
          ▼
AlertEvent 생성 (type_name='장시간개방', 중복 방지)
          │
          ▼
WebSocket broadcast (기존 alert.new 이벤트 포맷 동일)
```

**DB 폴링 방식은 채택하지 않습니다.** APScheduler long_open job 불필요.

---

### 구현 단계

#### 1. DB 마이그레이션 — EntityType에 threshold_minutes 추가

**파일**: `backend/app/models/entity_types.py`

```python
class EntityType(Base):
    __tablename__ = "entity_types"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(Text, unique=True, nullable=False)
    is_default: Mapped[int] = mapped_column(Integer, default=0)
    threshold_minutes: Mapped[int | None] = mapped_column(Integer, default=None)
    # None = 장시간개방 체크 비활성화
```

Alembic migration 생성 후 `seed_dev.py`에 기본값 반영:
- 출입문 (id=1): `threshold_minutes = 10`
- 냉장고 (id=2): `threshold_minutes = 5`
- 카운터 (id=3): `threshold_minutes = None` (체크 안 함)
- 기타 (id=4): `threshold_minutes = None`

---

#### 2. `publish_monitored_entities` — payload 형식 변경

**파일**: `backend/app/routers/entities.py` — `_publish_monitored_entities` 함수

현재: `entities: string[]` 발행  
변경: `entities: object[]` 발행 (ha_entity_id + threshold_minutes 포함)

```python
async def _publish_monitored_entities(store_id: int, db: AsyncSession) -> None:
    from app.models.entity_types import EntityType
    result = await db.execute(
        select(StoreEntity.ha_entity_id, EntityType.threshold_minutes)
        .outerjoin(EntityType, StoreEntity.type_id == EntityType.id)
        .where(StoreEntity.store_id == store_id, StoreEntity.is_active == 1)
    )
    entities = [
        {"ha_entity_id": row[0], "threshold_minutes": row[1]}
        for row in result.all()
    ]
    mqtt_client.publish_monitored_entities(store_id, entities)
```

**파일**: `backend/app/mqtt_client.py` — `publish_monitored_entities` 함수 시그니처 변경

```python
def publish_monitored_entities(store_id: int, entities: list[dict]):
    """entities: [{"ha_entity_id": "...", "threshold_minutes": int|None}, ...]"""
    if not _client:
        return
    payload = json.dumps({
        "store_id": store_id,
        "entities": entities,
        "timestamp": datetime.now(timezone.utc).isoformat(),
    })
    _client.publish(f"pcbang/{store_id}/config/monitored_entities", payload, qos=1, retain=True)
```

---

#### 3. MQTT 구독 추가 — long_open 토픽

**파일**: `backend/app/mqtt_client.py` — `_on_connect` 함수

```python
def _on_connect(client, userdata, flags, reason_code, *args):
    if reason_code == 0:
        logger.info("MQTT connected")
        client.subscribe("pcbang/+/status", qos=1)
        client.subscribe("pcbang/+/entities/+/state", qos=1)
        client.subscribe("pcbang/+/response/entities", qos=1)
        client.subscribe("pcbang/+/entities/+/long_open", qos=1)  # 추가
```

`_on_message` 함수에 라우팅 추가:

```python
# pcbang/{store_id}/entities/{ha_entity_id}/long_open
elif len(parts) == 5 and parts[2] == "entities" and parts[4] == "long_open":
    try:
        store_id = int(parts[1])
    except ValueError:
        return
    ha_entity_id = parts[3]
    asyncio.run_coroutine_threadsafe(
        _handle_long_open(store_id, ha_entity_id, payload), _loop
    )
```

---

#### 4. `_handle_long_open` 핸들러 신규 작성

**파일**: `backend/app/mqtt_client.py`

```python
async def _handle_long_open(store_id: int, ha_entity_id: str, payload: dict):
    from app.database import AsyncSessionLocal
    from app.models.alert_events import AlertEvent
    from app.models.store_entities import StoreEntity
    from app.models.stores import Store
    from app.models.monitoring_schedules import MonitoringSchedule
    from app.services.alert import _is_in_schedule
    from sqlalchemy import select, and_

    duration_minutes = payload.get("duration_minutes", 0)

    async with AsyncSessionLocal() as db:
        # entity 조회
        entity_result = await db.execute(
            select(StoreEntity).where(
                StoreEntity.store_id == store_id,
                StoreEntity.ha_entity_id == ha_entity_id,
            )
        )
        entity = entity_result.scalar_one_or_none()
        if not entity:
            return

        # 중복 방지: 같은 entity에 미확인 장시간개방 알림이 있으면 생성 안 함
        dup_result = await db.execute(
            select(AlertEvent).where(
                AlertEvent.store_id == store_id,
                AlertEvent.ha_entity_id == ha_entity_id,
                AlertEvent.type_name == "장시간개방",
                AlertEvent.acknowledged_by == None,
            )
        )
        if dup_result.scalar_one_or_none():
            logger.info("long_open 중복 알림 무시: store=%s entity=%s", store_id, ha_entity_id)
            return

        # 매장 조회
        store_result = await db.execute(select(Store).where(Store.store_id == store_id))
        store = store_result.scalar_one_or_none()
        if not store:
            return

        # is_in_schedule 판단
        from datetime import datetime, timezone as tz
        now = datetime.now(tz.utc)
        sched_result = await db.execute(
            select(MonitoringSchedule).where(MonitoringSchedule.store_id == store_id)
        )
        schedules = sched_result.scalars().all()
        in_schedule = _is_in_schedule(store, schedules, now)

        # stream_url
        stream_url: str | None = None
        if entity.camera_channel and store.go2rtc_url:
            from app.models.cameras import Camera
            cam_result = await db.execute(
                select(Camera).where(Camera.store_id == store_id, Camera.channel == entity.camera_channel)
            )
            cam = cam_result.scalar_one_or_none()
            if cam and cam.stream_source:
                stream_url = f"{store.go2rtc_url}/stream.html?src={cam.stream_source}"

        # AlertEvent 저장
        alert = AlertEvent(
            store_id=store_id,
            ha_entity_id=ha_entity_id,
            type_name="장시간개방",
            custom_name=entity.custom_name,
            state_from="on",
            state_to="on",
            stream_url=stream_url,
            importance=store.importance,
            is_in_schedule=1 if in_schedule else 0,
            occurred_at=now,
        )
        db.add(alert)
        await db.commit()
        await db.refresh(alert)

    entity_name = entity.custom_name or ha_entity_id
    _safe_broadcast({
        "type": "alert.new",
        "alert_id": alert.id,
        "store_id": store_id,
        "store_name": store.name,
        "importance": store.importance,
        "ha_entity_id": ha_entity_id,
        "entity_name": entity_name,
        "custom_name": entity.custom_name,
        "type_name": "장시간개방",
        "state_from": "on",
        "state_to": "on",
        "message": f"{entity_name} 장시간 개방 ({duration_minutes}분)",
        "stream_url": stream_url,
        "is_in_schedule": in_schedule,
        "timestamp": now.isoformat(),
    })
```

---

## B1-BE — 매장 검색 + 페이지네이션 API

**파일**: `backend/app/routers/stores.py`

`GET /stores` 에 쿼리 파라미터 추가:

```python
from fastapi import Query
from sqlalchemy import func, or_

@router.get("/stores")
async def list_stores(
    q: str | None = Query(default=None, description="매장명/주소 검색"),
    page: int = Query(default=1, ge=1),
    limit: int = Query(default=100, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
    _user: User = Depends(get_current_user),
):
    base_q = select(Store)
    count_q = select(func.count()).select_from(Store)

    if q:
        pattern = f"%{q}%"
        filter_expr = or_(
            Store.name.ilike(pattern),
            Store.address.ilike(pattern),
        )
        base_q = base_q.where(filter_expr)
        count_q = count_q.where(filter_expr)

    total = (await db.execute(count_q)).scalar_one()

    base_q = base_q.order_by(Store.importance.desc(), Store.name).offset((page - 1) * limit).limit(limit)
    stores = (await db.execute(base_q)).scalars().all()

    return {
        "success": True,
        "data": {
            "items": [StoreOut.model_validate(s) for s in stores],
            "total": total,
            "page": page,
            "total_pages": max(1, (total + limit - 1) // limit),
        }
    }
```

> **주의**: 기존 `data: Store[]` → `data: { items, total, page, total_pages }` 응답 형식 변경.  
> 완료 후 `shared/to_frontend/from_backend_20260408_b1_api_ready.md` 발행 필수.

---

## 테스트 포인트

### A4-BE
- `pcbang/30584/entities/binary_sensor.xxx/long_open` 발행 시 AlertEvent 생성 확인
- `ha_entity_id` → AlertEvent 조회 — `type_name='장시간개방'` 확인
- 중복 방지: 미확인 장시간개방 알림 있을 때 재발행해도 새 AlertEvent 생성 안 됨
- Frontend WS에서 `alert.new` 이벤트 수신 확인

### B1-BE
- `GET /stores?q=강남` → 강남 포함 매장만 반환
- `GET /stores?page=1&limit=2` → 2개 반환, `total_pages` 정확
- `GET /stores` (파라미터 없음) → 기존처럼 전체 반환 (items로 래핑됨)

---

## 완료 후 통보

1. A4-BE 완료: `shared/to_edge/from_backend_20260408_a4_ready.md` → Edge에게 통합 테스트 요청
2. B1-BE 완료: `shared/to_frontend/from_backend_20260408_b1_api_ready.md` → API 스펙 최종본 포함
