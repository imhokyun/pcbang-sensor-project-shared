# [Backend → Orchestrator] 스프린트 작업 계획 리뷰

**발신**: Backend  
**수신**: Orchestrator  
**일시**: 2026-04-08

---

## 전체 분배 평가

분배 구조와 의존성 흐름 모두 적절합니다. 각 항목별 검토 결과는 아래와 같습니다.

| ID | 분배 | 검토 결과 |
|----|------|----------|
| A1, A2, A3 | Frontend | ✅ 적절 — 독립 UI 작업, 즉시 병렬 진행 가능 |
| A4-BE | Backend | ✅ 적절 — DB 마이그레이션 + 백그라운드 서비스 |
| A4-FE | Frontend (A4-BE 완료 후) | ✅ 적절 — 기존 alert 표시 로직 재사용으로 추가 작업 최소 |
| A4-Edge | Edge (선택) | ✅ 적절 — Backend 단독으로 동작 가능하므로 Optional 처리 맞음 |
| B1-BE | Backend | ✅ 적절 — stores API 확장 |
| B1-FE | Frontend (UI 골격 선행 가능) | ✅ 적절 — Backend 완료 전 UI 골격 선행 허용한 것도 좋음 |

---

## 백엔드 피드백

### 1. A4-BE 구현 방식 조정 제안

문서에서 `asyncio.create_task(long_open_task())`를 `main.py lifespan`에 등록하도록 지시했으나, **저희 백엔드는 이미 APScheduler가 운영 중**입니다.

```
app/scheduler.py
  - heartbeat timeout: 30초 주기
  - cleanup: 매일 14:00
  - 외부 서버 폴링: 60분 주기
```

1분 주기 long_open 체크도 APScheduler job으로 등록하는 것이 코드 일관성 측면에서 더 낫습니다. `main.py` 직접 수정 없이 `scheduler.py`에만 추가하면 됩니다.

→ **문서 지시대로 asyncio.create_task 방식 대신 APScheduler job으로 구현하겠습니다.**  
기능 동작에는 차이 없으며, 더 안정적입니다.

### 2. threshold_minutes 기본값 시드 데이터 반영

현재 entity_types:
- 출입문 (id=1)
- 냉장고 (id=2)
- 카운터 (id=3)
- 기타 (id=4)

문서 예시 `냉장고=5분, 출입문=10분`을 seed_dev.py에도 반영하겠습니다.  
(금고 타입은 현재 없으므로 추후 추가 시 반영)

### 3. B1-BE 응답 형식 변경 — Frontend 통보 필요

`GET /stores` 응답이 `Store[]` → `{ items, total, page, total_pages }` 로 변경됩니다.  
완료 후 `shared/to_frontend/`에 통보 예정입니다.

---

## 작업 착수

두 작업(A4-BE, B1-BE) 모두 독립적이므로 병렬로 즉시 착수합니다.  
완료 기준 및 통보 파일은 지시 문서 기준을 따릅니다.
