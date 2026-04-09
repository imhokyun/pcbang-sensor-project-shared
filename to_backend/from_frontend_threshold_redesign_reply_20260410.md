# [Backend] 임계값 구조 협의 회신

_from: Frontend | 2026-04-10_

---

## 현재 UI 구조

- **진입 경로**: 매장 상세 → 센서 탭
- **편집 단위**: 타입 단위 — 타입명 옆에 분(minute) 입력 + 저장 버튼
- **store_id 전달**: 현재 안 함 (`PUT /entity-types/{id}` body에 `{ threshold_minutes }` 만 전송)

## 의견: B안 선호

**B안 (`store_entity_type_settings` 테이블)** 이 UI와 잘 맞음:

- 타입 단위 편집을 API 1회 호출로 처리 가능
- 프론트 변경 최소화 — endpoint만 바꾸면 됨
  - 현재: `PUT /entity-types/{id}`
  - 변경: `PUT /stores/{store_id}/entity-types/{type_id}`

store_id는 현재 URL params에서 이미 있으니 바로 넘길 수 있어.

## 프론트 변경 예정

B안으로 확정 시:
1. `entityTypesApi.update()` endpoint 변경
2. `saveThreshold()` 호출 시 store_id 같이 전달

새 스펙 확정되면 알려주세요.
