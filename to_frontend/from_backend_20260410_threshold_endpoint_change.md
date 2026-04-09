# [Frontend] 장시간 개방 임계값 — 엔드포인트 변경 완료

_from: Backend | 2026-04-10_

---

## 변경 사항

B안으로 확정 및 구현 완료.

### 삭제된 엔드포인트
```
PUT /entity-types/{id}   ← 제거됨
```

### 신규 엔드포인트

**타입 목록 + 임계값 조회 (매장별)**
```
GET /stores/{store_id}/entity-types
```
응답:
```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 30.0 },
    { "id": 2, "name": "냉장고", "is_default": 1, "threshold_minutes": 5.0 }
  ]
}
```
> 임계값 미설정 타입은 기본값 5분으로 반환됩니다.

**임계값 수정 (매장별)**
```
PUT /stores/{store_id}/entity-types/{type_id}
```
요청:
```json
{ "threshold_minutes": 30 }
```
응답:
```json
{ "success": true, "data": { "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 30.0 } }
```
> null 전송 시 기본값 5분으로 초기화됩니다.
> 저장 후 해당 매장 Edge에 MQTT로 자동 재발행됩니다.

## 기존 GET /entity-types

유지되나 `threshold_minutes` 필드가 제거됩니다:
```json
{ "id": 1, "name": "출입문", "is_default": 1 }
```
entity 등록 시 타입 선택 드롭다운 용도로만 사용 권장.

## 프론트 변경 필요 항목

1. 매장 상세 센서 탭 진입 시: `GET /entity-types` → `GET /stores/{store_id}/entity-types` 로 변경
2. 임계값 저장: `PUT /entity-types/{id}` → `PUT /stores/{store_id}/entity-types/{type_id}` 로 변경
   - body에서 `store_id` 제거 (이제 URL path로 전달)
