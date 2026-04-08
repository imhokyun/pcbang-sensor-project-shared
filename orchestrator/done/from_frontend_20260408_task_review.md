# [Frontend → Orchestrator] 작업 지시 검토 및 문서 최신화 완료

**발신**: Frontend  
**수신**: Orchestrator  
**일시**: 2026-04-08

---

## 문서 최신화 완료

`shared/docs/frontend-screens.md`, `shared/docs/architecture.md`, `shared/contracts/api.md`를  
현재 구현된 코드베이스 기준으로 최신화했습니다.  
작업 지시(`from_orchestrator_20260408_tasks.md`) 검토 전에 **업데이트된 문서를 먼저 확인해 주시기 바랍니다.**

---

## 검토 의견

### A3, A2 — 이상 없음

현재 구현과 충돌 없습니다. 지시대로 진행 가능합니다.

---

### A1 — `alert.entity_name` 필드 불일치

지시서에서 팝업 UI 개선 시 아래 필드를 사용하도록 명시하고 있습니다:

```typescript
alert.entity_name   // 센서명
```

그러나 **현재 구현에서는 해당 필드가 없습니다.**  
`Alert` 타입 및 WS `alert.new` 이벤트 처리 코드 모두 `custom_name`을 사용합니다.

**현재 사용 중인 필드:**
```typescript
alert.custom_name   // 센서 표시명 (현재 사용)
alert.type_name     // 기기 타입
alert.store_name    // 매장명
alert.importance    // 중요도
```

`custom_name`은 관리자가 설정한 표시명이므로 센서명 표시 목적에 부합합니다.  
`entity_name` 필드가 별도로 필요한지, 아니면 `custom_name`으로 대체 가능한지 확인 후  
작업 지시를 재정리해 주시면 그에 따라 진행하겠습니다.

---

### A4-FE — 추가 개발 불필요 예상

A2(타입별 섹션 분리) 구현 시 `type_name='장시간개방'` 알림은 자동으로 해당 섹션에 표시됩니다.  
별도 개발 없이 Backend A4-BE 완료 후 실수신 테스트만 진행하면 될 것으로 판단합니다.  
Backend-Edge 간 구현 방식 협의 결과도 함께 공유 부탁드립니다.

---

### B1-FE — UI 골격 선행, API 연결은 Backend 완료 후

현재 `GET /stores` 응답이 `Store[]`인데, B1-BE 완료 후 `{ items, total, page, total_pages }`로 변경됩니다.  
**UI 골격(검색창, 페이지네이션 컴포넌트)은 선행 구현하겠습니다.**  
API 타입 연결은 Backend 완료 신호(`shared/to_frontend/`) 수신 후 진행합니다.

---

## 요청

위 내용 검토 후, 특히 **A1의 `entity_name` vs `custom_name` 방향**을 확정하여  
작업 지시를 재정리해 주시면 감사합니다.
