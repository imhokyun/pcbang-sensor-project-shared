# [Backend → Frontend] B1-BE 완료 — GET /stores 응답 형식 변경

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-08

---

## 변경 내용

`GET /stores` 응답 형식이 변경되었습니다.

### 변경 전
```json
{
  "success": true,
  "data": [ { ...Store }, { ...Store } ]
}
```

### 변경 후
```json
{
  "success": true,
  "data": {
    "items": [ { ...Store }, { ...Store } ],
    "total": 2,
    "page": 1,
    "total_pages": 1
  }
}
```

---

## 추가된 쿼리 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|--------|------|
| `q` | string | null | 매장명/주소 검색 (ILIKE, 부분 일치) |
| `page` | int | 1 | 페이지 번호 (1부터) |
| `limit` | int | 100 | 페이지 크기 (최대 100) |

**예시:**
```
GET /stores?q=강남          → 강남 포함 매장만 반환
GET /stores?page=1&limit=10 → 첫 10개
GET /stores                 → 전체 (items로 래핑)
```

---

## 조치 요청

`storesApi.list()` 호출 후 `data` 배열을 직접 사용하는 코드를 `data.items`로 수정해야 합니다.

```typescript
// 변경 전
const stores = response.data.data  // Store[]

// 변경 후
const stores = response.data.data.items  // Store[]
const total = response.data.data.total
const totalPages = response.data.data.total_pages
```
