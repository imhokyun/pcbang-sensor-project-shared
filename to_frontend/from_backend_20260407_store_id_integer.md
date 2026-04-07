# store_id 타입 변경 알림 — TEXT → INTEGER

**발신**: backend | **수신**: frontend | **날짜**: 2026-04-07

---

## 변경 내용

`store_id` 필드가 **TEXT("store_001")에서 INTEGER(30584)로 변경**되었습니다.

외부 관제관리서버가 매장을 숫자형 `storeNo`(예: 30584, 30585)로 관리하고 있어, 이에 맞게 통일합니다.

---

## 영향 범위

### API 요청/응답

| 항목 | 기존 | 변경 후 |
|---|---|---|
| `store_id` 타입 | `string` | `integer` |
| URL 경로 파라미터 | `/stores/store_001` | `/stores/30584` |
| POST /stores body | `"store_id": "store_001"` | `"store_id": 30584` |
| 응답 내 `store_id` | `"store_001"` | `30584` |

### WebSocket `init` / 이벤트

모든 WS 메시지의 `store_id` 필드도 integer로 반환됩니다:
```json
{ "type": "store.status", "store_id": 30584, "status": "online" }
{ "type": "entity.update", "store_id": 30584, ... }
{ "type": "alert.new", "store_id": 30584, ... }
```

---

## 개발 시드 데이터 (즉시 사용 가능)

| store_id | 매장명 | 상태 |
|---|---|---|
| `30584` | 강남점 | online |
| `30585` | 홍대점 | offline |

계정: `admin1` ~ `admin5` / 비밀번호: `admin1234`

---

## 조치 사항

- API 호출 시 `store_id`를 숫자로 전송
- 화면 표시 시 숫자로 렌더링 (또는 별도 매장명으로 표시)
- localStorage/상태관리에서 store_id를 number 타입으로 저장
