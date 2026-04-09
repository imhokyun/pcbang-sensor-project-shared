# [Backend] entity-types PUT 500 에러 + CORS 미적용

_from: Frontend | 2026-04-10_

---

## 1. CORS 여전히 미적용

어제 요청한 서버 재시작이 아직 안 된 것 같음.  
`https://pcbang-iot-api.multion.synology.me` 모든 요청에서 CORS 에러 발생 중.

백엔드 포트 8080 프로세스 재시작 부탁드립니다.

---

## 2. PUT /entity-types/{id} → 500

### 증상

매장 상세 > 센서 탭에서 장시간 개방 임계값 수정 시 500 에러 발생:

```
PUT https://pcbang-iot-api.multion.synology.me/api/v1/entity-types/1 → 500
```

### 프론트 요청 페이로드

```json
{ "threshold_minutes": 30 }
```

(null 허용 — 임계값 제거 시 null 전송)

### 계약 문서 현황

`shared/contracts/api.md`의 entity-types 섹션에 PUT 엔드포인트가 없음:

```
GET    /entity-types
POST   /entity-types
DELETE /entity-types/{id}
```

### 요청

1. `PUT /entity-types/{id}` 구현 또는 버그 수정
2. 계약 문서(`contracts/api.md`)에 PUT 스펙 추가

**요청 바디:**
```json
{ "threshold_minutes": number | null }
```

**응답:** 수정된 EntityType 객체
