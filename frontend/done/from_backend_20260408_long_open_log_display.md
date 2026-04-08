# [Backend → Frontend] 장시간개방 이벤트 로그 표시 안내

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-08

---

## 변경 사항

장시간개방 감지 시 이제 `alert_events`(알림)뿐 아니라 `sensor_events`(이벤트 로그)에도 기록됩니다.  
**DB 스키마 변경 없음** — 기존 `GET /logs` API 그대로 사용합니다.

---

## 로그 데이터 형식

`GET /logs` 응답에서 장시간개방 항목은 아래와 같이 옵니다:

```json
{
  "id": 42,
  "store_id": 30584,
  "store_name": "강남점",
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "custom_name": "냉장고 1",
  "type_name": "장시간개방",
  "state_from": "on",
  "state_to": "on",
  "occurred_at": "2026-04-08T14:32:00Z"
}
```

**일반 상태 변화 항목과의 차이:**

| 필드 | 일반 이벤트 | 장시간개방 |
|------|------------|-----------|
| `type_name` | `"출입문"`, `"냉장고"` 등 | `"장시간개방"` |
| `state_from` | `"off"` | `"on"` |
| `state_to` | `"on"` | `"on"` |

---

## 표시 가이드

`state_from → state_to` 형식 그대로 보여주면 **"on → on"** 이 되어 의미가 없습니다.  
`type_name === "장시간개방"` 일 때는 별도로 처리해 주세요.

### 권장 표시 방식

```
[일반] 냉장고 1   off → on     2026-04-08 14:30
[장시간] 냉장고 1  장시간 개방   2026-04-08 14:32
```

### 구현 예시 (TypeScript)

```typescript
function formatStateChange(log: LogItem): string {
  if (log.type_name === "장시간개방") {
    return "장시간 개방";
  }
  return `${log.state_from ?? "-"} → ${log.state_to}`;
}

function getTypeBadgeStyle(typeName: string): string {
  if (typeName === "장시간개방") return "badge-warning";  // 노란색 등 구분
  return "badge-default";
}
```

### 필터링

`GET /logs?type_name=장시간개방` 으로 장시간개방 이벤트만 조회 가능합니다.

---

## 요약

- API 변경 없음, `GET /logs` 그대로 사용
- `type_name === "장시간개방"` 조건으로 분기
- `state_from / state_to` 대신 `"장시간 개방"` 텍스트로 표시
