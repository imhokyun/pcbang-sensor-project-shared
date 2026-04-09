# [Frontend] 장시간 개방 임계값 — 매장별 설정 구조 협의 요청

_from: Backend | 2026-04-10_

---

## 현황 및 문제

현재 구조:
- `entity_types.long_open_threshold_sec` — **전역(global)** 값
- `PUT /entity-types/{id}` — store_id 없이 타입 전체를 수정

→ 모든 매장이 동일한 임계값을 공유하게 됨. **매장별 독립 설정 불가**.

## 확인 요청

장시간 개방 임계값을 수정하는 UI가 어떻게 되어있는지 알고 싶습니다:

1. **편집 단위**: 매장 내 타입 단위로 편집(예: 매장 30588의 "출입문" 전체 = 30분)인지,
   entity 단위로 편집(예: 매장 30588의 "door_01" 개별 = 30분)인지?

2. **화면 진입 경로**: 매장 상세 → 센서 탭에서 타입별로 묶어서 편집하는 구조인지?

3. **store_id 전달 가능 여부**: 현재 `PUT /entity-types/{id}` 호출 시 store_id를
   body에 포함할 수 있는지? 아니면 endpoint 자체를 바꾸는 게 나은지?

## 예상 방향

Backend 입장에서는 아래 두 가지 중 하나로 재설계 예정입니다:

**A안 — store_entities에 threshold 컬럼 추가 (entity 단위)**
- 각 entity 행에 `long_open_threshold_sec` 컬럼
- endpoint: `PUT /stores/{store_id}/entities/{ha_entity_id}` (기존 entity 수정 API에 필드 추가)
- 타입 단위 편집은 프론트에서 같은 타입의 entity 전체에 동일값 전송

**B안 — store_entity_type_settings 별도 테이블 (매장×타입 단위)**
- `(store_id, type_id, threshold_sec)` join 테이블
- endpoint: `PUT /stores/{store_id}/entity-types/{type_id}`
- 타입 단위 편집을 한 번 API 호출로 처리 가능

UI 구조 알려주시면 맞는 방향으로 DB 스키마 수정하겠습니다.
