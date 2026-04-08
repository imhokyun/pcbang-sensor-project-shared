# [Frontend → Backend] EntityType threshold_minutes API 노출 요청

**발신**: Frontend  
**수신**: Backend  
**일시**: 2026-04-08

---

## 요청 배경

A4 장시간개방 알림이 Backend·Edge 모두 구현 완료됐습니다.  
현재 `threshold_minutes`는 DB에 고정값으로만 존재하며, 관리자가 UI에서 조회·수정할 수 없습니다.  
관리자가 매장 환경에 맞게 threshold를 직접 설정할 수 있도록 API 노출을 요청합니다.

---

## 요청 사항

### 1. `GET /entity-types` 응답에 `threshold_minutes` 필드 추가

**현재 응답:**
```json
[
  { "id": 1, "name": "출입문", "is_default": 1 },
  { "id": 2, "name": "냉장고", "is_default": 1 }
]
```

**요청 응답:**
```json
[
  { "id": 1, "name": "출입문", "is_default": 1, "threshold_minutes": 10 },
  { "id": 2, "name": "냉장고", "is_default": 1, "threshold_minutes": 5 },
  { "id": 3, "name": "카운터", "is_default": 1, "threshold_minutes": null },
  { "id": 4, "name": "기타",   "is_default": 1, "threshold_minutes": null }
]
```

> `null` = 장시간개방 감지 비활성화

---

### 2. `POST /entity-types` 요청 body에 `threshold_minutes` 포함 허용

사용자 정의 타입 추가 시 threshold 지정 가능하도록:

```json
{ "name": "금고", "threshold_minutes": 3 }
```

> `threshold_minutes` 생략 또는 `null` → 감지 비활성화

---

### 3. `PUT /entity-types/{id}` 엔드포인트 신규 추가 (선택)

기존 타입의 threshold만 변경할 수 있는 엔드포인트:

```
PUT /entity-types/{id}
```

**request body:**
```json
{ "threshold_minutes": 3 }
```

> is_default 타입도 threshold_minutes는 수정 가능해야 합니다.  
> (이름 변경·삭제는 기본 타입에서 불가, threshold만 허용)

---

## Frontend 작업 계획

API 업데이트 완료 통보 수신 후:
- `TabEntities.tsx` EntityType 관리 섹션에 threshold_minutes 입력 필드 추가
- 표시 형식: `N분` 숫자 입력 (null = 비활성 체크박스 또는 빈칸)

---

## 우선순위

`PUT /entity-types/{id}` 는 선택사항이며, **GET 응답 필드 추가 + POST 허용이 최우선**입니다.  
완료 후 `shared/to_frontend/`에 통보 부탁드립니다.
