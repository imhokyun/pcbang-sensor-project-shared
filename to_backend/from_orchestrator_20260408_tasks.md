# Backend 작업 지시

**발신**: Orchestrator | **날짜**: 2026-04-08 | **브랜치**: dev/backend  
**전체 그림**: `shared/new_request_20260408.md` 먼저 읽어볼 것

---

## 담당 작업 요약

| ID | 작업 | 의존성 |
|----|------|--------|
| A4-BE | 장시간 개방 감지 백그라운드 태스크 + AlertEvent 생성 | 없음 |
| B1-BE | 매장 API 검색·페이지네이션 쿼리 파라미터 추가 | 없음 |

두 작업 모두 독립적이므로 **병렬 처리** 가능합니다.

---

## B1-BE — 매장 검색 + 페이지네이션 API

**파일**: `app/routers/stores.py` (또는 현재 stores 라우터 위치)

### 현재 상태
`GET /stores` — 파라미터 없이 전체 반환 (`storesApi.list()` 호출).

### 변경 사항

```python
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

    base_q = base_q.order_by(Store.name).offset((page - 1) * limit).limit(limit)
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

### 주의사항
- 기존 응답 형식 `Store[]` 에서 `{ items, total, page, total_pages }` 로 변경됩니다.
- Frontend 팀에도 이 변경을 통보해주세요 (작업 완료 시 `shared/to_frontend/` 에 파일 추가).
- `limit` 최대값 100 하드캡 (`le=100`).

---

## A4-BE — 장시간 개방 감지 백그라운드 태스크

### 설계

```
EntityType 테이블에 threshold_minutes 컬럼 추가
  (냉장고=5, 출입문=10, 금고=2, 기타=15, 기본값=None이면 체크 안 함)
          ↓
app/services/long_open.py (신규)
  - check_long_open_sensors(db) 함수
  - SensorEvent에서 각 entity별 마지막 state_to='on' 시각 조회
  - 현재시각 - 해당 시각 > threshold_minutes → AlertEvent 생성
  - 이미 같은 entity에 대해 미확인(acknowledged=False) 장시간개방 알림이 있으면 중복 생성 안 함
          ↓
app/main.py 에 lifespan 태스크 등록
  - 1분 주기 asyncio loop
          ↓
WebSocket으로 기존 alert_new 이벤트와 동일하게 전송
```

### 단계별 구현

**1. DB 마이그레이션 — EntityType에 threshold_minutes 추가**
```python
# alembic migration
threshold_minutes: Mapped[int | None] = mapped_column(Integer, default=None)
# None이면 장시간개방 체크 비활성화
```

**2. `app/services/long_open.py` 신규 생성**
```python
async def check_long_open_sensors(db: AsyncSession) -> list[AlertEvent]:
    """
    현재 'on' 상태이고 threshold_minutes 이상 지속된 entity 탐지.
    이미 미확인 장시간개방 알림이 있는 entity는 건너뜀.
    """
    # 각 entity의 마지막 SensorEvent (state_to='on') 조회
    # StoreEntity → EntityType join으로 threshold_minutes 가져옴
    # (현재시각 - occurred_at).total_seconds() / 60 > threshold_minutes
    # AlertEvent 생성: type_name='장시간개방', importance=entity 중요도 또는 3
    ...
```

**3. `app/main.py` — lifespan에 백그라운드 태스크 등록**
```python
async def long_open_task():
    while True:
        await asyncio.sleep(60)
        async with AsyncSessionLocal() as db:
            try:
                new_alerts = await check_long_open_sensors(db)
                for alert in new_alerts:
                    await broadcast_alert_new(alert)  # 기존 WS 브로드캐스트 함수
            except Exception as e:
                logger.error(f"long_open_task error: {e}")

# lifespan context 내 asyncio.create_task(long_open_task())
```

**4. AlertEvent.type_name 값**: `'장시간개방'` (한글, 기존 타입들과 일관성 유지)

### 완료 후 통보

작업 완료 시:
1. `shared/to_frontend/from_backend_20260408_b1_api_ready.md` — B1 API 스펙 최종 확정본
2. `shared/to_edge/from_backend_20260408_a4_spec.md` — A4 장시간개방 알림 스펙 (Edge 선택 보완용)

---

## 테스트 포인트

- `GET /stores?q=강남` → 강남점만 반환
- `GET /stores?page=1&limit=2` → 2개 반환, total_pages 계산 정확
- 센서 state_to='on' 이후 threshold_minutes 경과 → AlertEvent 생성 확인
- 중복 알림 방지: 같은 entity에 미확인 장시간개방 알림 있으면 신규 생성 안 함
