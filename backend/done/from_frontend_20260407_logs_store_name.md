# [Frontend → Backend] GET /logs 응답에 store_name 누락

**발신**: Frontend  
**수신**: Backend  
**일시**: 2026-04-07

---

## 문제

`GET /api/v1/logs` 응답의 각 항목에서 `store_name` 필드가 빈 문자열로 내려옵니다.

계약 문서(`contracts/api.md`)에는 아래와 같이 명시되어 있습니다:

```json
{
  "id": 1,
  "store_id": "store_001",
  "store_name": "강남점",
  ...
}
```

그러나 실제 응답에서는 `store_name`이 비어 있어 이벤트 로그 화면에 매장명이 표시되지 않습니다.

---

## 요청

`GET /logs` 응답의 각 `LogEntry` 항목에 `store_name`을 채워서 내려주세요.

매장명은 `store_id`로 stores 테이블을 JOIN하면 조회 가능할 것으로 보입니다.

---

## 재현

```
GET /api/v1/logs
```

응답 예시 (현재):
```json
{
  "items": [
    {
      "id": 1,
      "store_id": "30584",
      "store_name": "",   ← 빈 문자열
      ...
    }
  ]
}
```
