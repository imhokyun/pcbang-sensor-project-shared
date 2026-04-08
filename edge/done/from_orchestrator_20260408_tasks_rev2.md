# Edge 작업 지시 (Rev.2)

**발신**: Orchestrator | **날짜**: 2026-04-08 | **브랜치**: dev/edge  
**전체 그림**: `shared/new_request_20260408_rev2.md` 먼저 읽어볼 것

> **Rev.1 대비 변경**: A4-Edge가 Optional → **이번 스프린트 필수**로 격상

---

## 담당 작업 요약

| ID | 작업 | 의존성 | 우선순위 |
|----|------|--------|---------|
| A4-Edge | coordinator.py에 asyncio 타이머 + long_open MQTT 발행 | Backend A4-BE와 병렬 가능 | **필수** |

---

## A4-Edge — 장시간 개방 asyncio 타이머 구현

### 배경

Backend·Edge 양 팀이 건의한 "Edge 감지 방식"이 승인되었습니다.

HA 내장 MQTT integration이 비활성화된 환경이므로, **옵션 2(Custom Component 타이머)** 방식으로 구현합니다.  
`coordinator.py`에 asyncio 타이머 로직 추가 — 별도 파일 불필요.

---

### 구현 내용

**파일**: `edge/homeassistant/custom_components/pcbang_sensor/coordinator.py`

#### 1. `__init__` — 타이머 상태 필드 추가

```python
def __init__(self, hass: HomeAssistant, store_id: int, mqtt_client: MQTTClient) -> None:
    self._hass = hass
    self._store_id = store_id
    self._mqtt = mqtt_client
    self._monitored_entities: set[str] = set()
    self._entity_thresholds: dict[str, int] = {}        # ha_entity_id → threshold_minutes
    self._long_open_timers: dict[str, asyncio.Task] = {} # ha_entity_id → 실행 중인 타이머 Task
    self._unsub_state_listener = None
```

---

#### 2. `teardown` — 타이머 정리

```python
def teardown(self) -> None:
    if self._unsub_state_listener:
        self._unsub_state_listener()
        self._unsub_state_listener = None
    # 모든 실행 중인 타이머 취소
    for task in self._long_open_timers.values():
        task.cancel()
    self._long_open_timers.clear()
```

---

#### 3. `handle_monitored_entities` — object[] 형식 파싱으로 변경

현재: `entities: string[]` 파싱
변경: `entities: object[]` 파싱 (Backend Rev.2 완료 후 적용됨)

```python
def handle_monitored_entities(self, topic: str, payload: dict) -> None:
    """pcbang/{store_id}/config/monitored_entities 수신."""
    entities = payload.get("entities", [])

    # 새 형식: object[] (ha_entity_id + threshold_minutes)
    # 기존 형식: string[] — 하위 호환 유지
    if entities and isinstance(entities[0], str):
        # 구형 payload (Backend 업데이트 전 임시 호환)
        self._monitored_entities = set(entities)
        self._entity_thresholds = {}
        _LOGGER.info("[Coordinator] monitored_entities (legacy format): %d개", len(self._monitored_entities))
    else:
        # 신형 payload
        self._monitored_entities = {e["ha_entity_id"] for e in entities}
        self._entity_thresholds = {
            e["ha_entity_id"]: e["threshold_minutes"]
            for e in entities
            if e.get("threshold_minutes") is not None
        }
        _LOGGER.info(
            "[Coordinator] monitored_entities: %d개, threshold 설정: %d개",
            len(self._monitored_entities),
            len(self._entity_thresholds),
        )

    # monitored_entities 변경 시 기존 타이머 모두 취소 (목록 바뀌었으므로)
    for task in self._long_open_timers.values():
        task.cancel()
    self._long_open_timers.clear()
```

---

#### 4. `handle_state_changed` — 타이머 시작/취소 로직 추가

```python
def handle_state_changed(
    self,
    entity_id: str,
    old_state: State | None,
    new_state: State,
) -> None:
    """entity 상태 변화 처리 — monitored 대상만 publish + 타이머 관리."""
    if entity_id not in self._monitored_entities:
        return

    old = old_state.state if old_state else "없음"
    _LOGGER.info("[Coordinator] state_changed 발행: %s  %s → %s", entity_id, old, new_state.state)

    # 상태 MQTT publish (기존 유지)
    self._mqtt.publish(
        TOPIC_ENTITY_STATE.format(store_id=self._store_id, ha_entity_id=entity_id),
        {
            "store_id": self._store_id,
            "ha_entity_id": entity_id,
            "state": new_state.state,
            "timestamp": _now_iso(),
        },
        qos=1,
        retain=True,
    )

    # 장시간 개방 타이머 관리
    threshold = self._entity_thresholds.get(entity_id)

    if new_state.state == "on" and threshold is not None:
        # 기존 타이머 취소 후 새 타이머 시작
        existing = self._long_open_timers.get(entity_id)
        if existing and not existing.done():
            existing.cancel()
        task = self._hass.loop.create_task(
            self._long_open_timer(entity_id, threshold)
        )
        self._long_open_timers[entity_id] = task
        _LOGGER.info("[Coordinator] long_open 타이머 시작: %s (%d분)", entity_id, threshold)

    elif new_state.state == "off":
        # 타이머 취소
        existing = self._long_open_timers.pop(entity_id, None)
        if existing and not existing.done():
            existing.cancel()
            _LOGGER.info("[Coordinator] long_open 타이머 취소: %s (off 전환)", entity_id)
```

---

#### 5. `_long_open_timer` — 타이머 코루틴 신규 추가

```python
async def _long_open_timer(self, entity_id: str, threshold_minutes: int) -> None:
    """threshold_minutes 경과 후 long_open MQTT 발행."""
    try:
        await asyncio.sleep(threshold_minutes * 60)
    except asyncio.CancelledError:
        return  # 정상 취소 (off 전환 등)

    # 타이머 만료 — 현재도 on 상태인지 확인 (선택적 안전장치)
    current = self._hass.states.get(entity_id)
    if current is None or current.state != "on":
        _LOGGER.debug("[Coordinator] long_open 만료 무시 (현재 off): %s", entity_id)
        self._long_open_timers.pop(entity_id, None)
        return

    _LOGGER.info("[Coordinator] long_open 발행: %s (%d분 경과)", entity_id, threshold_minutes)
    self._mqtt.publish(
        f"pcbang/{self._store_id}/entities/{entity_id}/long_open",
        {
            "ha_entity_id": entity_id,
            "duration_minutes": threshold_minutes,
            "triggered_at": _now_iso(),
        },
        qos=1,
        retain=False,
    )
    # 타이머 dict에서 제거 (완료)
    self._long_open_timers.pop(entity_id, None)
```

---

## MQTT 토픽 확정 (contracts/mqtt.md 참조)

```
Topic:   pcbang/{store_id}/entities/{ha_entity_id}/long_open
Payload: {
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
QoS: 1, retain: false
```

---

## 구현 시 주의사항

1. **asyncio.CancelledError 반드시 catch** — `sleep` 취소 시 누락되면 로그 오류 발생
2. **`hass.loop.create_task`** 사용 — HA의 event loop에서 실행해야 함
3. **monitored_entities 변경 시 타이머 리셋** — 이미 구현에 포함됨
4. **HA 재시작 시**: monitored_entities는 retain=true이므로 자동 재수신 → 타이머는 재시작됨  
   단, 재시작 전에 이미 on 상태인 entity의 타이머는 재설정되지 않음 (state_changed 이벤트 없으므로)  
   → 허용 범위 내 (재시작 후 on→off→on 사이클에서 재등록)

---

## 테스트 포인트

Backend A4-BE 완료 통보 수신 후:

1. HA에서 `binary_sensor` 수동으로 `on` 상태로 설정
2. `threshold_minutes` (냉장고=5분, 출입문=10분) 경과
3. MQTT 브로커에서 `pcbang/{store_id}/entities/{ha_entity_id}/long_open` 발행 확인
4. Backend에서 AlertEvent 생성 확인
5. Frontend 알림 화면에 "장시간개방" 알림 표시 확인

---

## 완료 후 통보

`shared/to_orchestrator/from_edge_20260408_a4_complete.md` 에 테스트 결과 리포트
