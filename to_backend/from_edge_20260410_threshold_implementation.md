# [Backend] 장시간 개방 임계값 구현 상태 회신

_from: Edge | 2026-04-10_

---

## 질문 1 — threshold_minutes 실제 구현 여부

**구현 완료.** `coordinator.py`에서 `_entity_thresholds: dict[str, int]`에 저장 후 asyncio 타이머로 관리 중입니다.

동작 흐름:
```
monitored_entities 수신
  → _monitored_entities (set), _entity_thresholds (dict) 메모리 업데이트

state_changed 발생 → entity가 monitored이고 state == "on"
  → threshold 있으면 asyncio.sleep(threshold_minutes * 60) 타이머 시작
  → 타이머 만료 시 현재도 "on"인지 재확인 후 long_open 발행
    topic: pcbang/{store_id}/entities/{ha_entity_id}/long_open
    payload: {"ha_entity_id": "...", "duration_minutes": N, "triggered_at": "..."}
```

---

## 질문 2 — null 처리

**비활성 처리됨.** `threshold_minutes: null`인 entity는 `_entity_thresholds`에 등록되지 않으므로 타이머가 시작되지 않습니다. state 변화 publish는 정상 동작합니다.

```python
# coordinator.py 발췌
self._entity_thresholds = {
    e["ha_entity_id"]: e["threshold_minutes"]
    for e in entities
    if e.get("threshold_minutes") is not None  # null 제외
}
```

---

## 질문 3 — 재수신 시 타이머 동작

**기존 타이머 즉시 전체 취소 → 새 값은 다음 state 변경부터 적용.**

`monitored_entities`가 재수신되면:
1. 진행 중인 모든 `_long_open_timers` task 즉시 cancel
2. 새 `_entity_thresholds` 적용
3. **현재 "on" 상태인 entity에 대해 타이머를 즉시 재시작하지는 않음** — 다음 state_changed 이벤트 시점에 새 threshold 값으로 시작됨

> 만약 "threshold 변경 즉시 재시작" 동작이 필요하다면 알려주세요. `monitored_entities` 수신 시 현재 "on" 상태인 monitored entity를 조회해 타이머를 바로 시작하도록 수정 가능합니다.

---

## Edge 측 변경사항

없음. 현재 payload 구조(`entities: [{ha_entity_id, threshold_minutes}]`)로 이미 동작 중이므로 Backend DB 재설계 후에도 Edge 수정 없이 그대로 연동됩니다.
