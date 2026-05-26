# Frontend → CTRL 동기화 작업 지시 (Phase 2)

발신: orchestrator (main)
일시: 2026-05-26
원본 plan: `shared/orchestrator/design_20260526_ctrl_sync.md`
계약 갱신: `shared/contracts/api.md` §CTRL 동기화 (POST /stores/{id}/schedules/sync 응답 형식)

---

## 작업 방식 — behavior-first

자동 테스트 infra 없는 상태 유지 (F-TEST-INFRA는 별도 후속).

1. **시나리오 명시** (commit body draft)
2. **agent-browser before 스크린샷** — 변경 전 매장 상세 페이지
3. **구현**
4. **agent-browser after 스크린샷** — 성공 토스트 + 에러 케이스 둘 다
5. **회귀 둘러보기** — 매장 상세의 다른 탭(기본정보·CCTV·센서) 위화감 X 확인

---

## Task

### T6 FE-BUTTON — "외부 서버에서 동기화" 버튼

**시나리오**:
- `/stores/{store_id}` 페이지의 "관제시간 설정" 탭 또는 그 근처에 **[외부 서버에서 동기화]** 버튼
- 클릭 → loading 표시 → `POST /api/v1/stores/{store_id}/schedules/sync` 호출
- 응답 200 → 성공 토스트 `"동기화 완료: X일 갱신, Y건 수동 설정 보존"` (응답 `data.updated_count`, `data.skipped_manual` 활용)
- 응답 502 `CTRL_AUTH_FAILED` → 에러 토스트 `"외부 인증 실패. 운영팀에 문의하세요"`
- 응답 504 `CTRL_TIMEOUT` → 에러 토스트 `"외부 서버 응답 없음. 잠시 후 다시 시도해주세요"`
- 응답 502 `CTRL_STORE_NOT_FOUND` → 에러 토스트 `"CTRL에서 이 매장을 찾을 수 없습니다"`
- 응답 404 `STORE_NOT_FOUND` (우리 DB 미존재) → 에러 토스트 `"매장 정보가 우리 시스템에 없습니다"`
- 그 외 5xx → "동기화 실패. 다시 시도해주세요"
- 버튼은 호출 중 disabled, 응답 후 자동 enable

**구현**:

1. `frontend/lib/api.ts` — 함수 추가:
   ```ts
   export interface SyncSchedulesResult {
     updated_count: number;
     skipped_manual: number;
   }
   export async function syncStoreSchedules(storeId: number): Promise<SyncSchedulesResult> {
     // POST /api/v1/stores/{storeId}/schedules/sync
     // ApiError로 502/504/404 메시지 매핑
   }
   ```

2. `frontend/app/stores/[store_id]/page.tsx` — 버튼 + 다이얼로그 없이 즉시 호출
   - 위치: 관제시간 탭 헤더 우측. 또는 매장 상세 페이지 상단 액션 영역.
   - 라벨: `↻ 외부 서버에서 동기화`
   - 토스트는 기존 토스트 메커니즘(`StoreGrid.tsx`의 setToasts 패턴 참고) 또는 새 토스트 함수
   - 로딩 중 spinner + `동기화 중...` 라벨

**검증** (agent-browser):
- 1. `/stores/30584` 진입 → 버튼 보임
- 2. 버튼 클릭 → 로딩 → 토스트 성공 메시지 캡처
- 3. backend mock 또는 .env에서 CTRL_API_KEY를 잘못된 값으로 (사용자 dry-run 단계) → 502 에러 토스트 캡처
- 4. 매장 상세의 다른 탭/섹션 회귀 — 시각·동작 변함 없음 캡처

**커밋**: `[T6-FE-CTRL] stores/{id} 외부 서버 동기화 버튼 + 토스트`

---

## 진행 절차

```bash
cd /home/multion/production/pcbang-sensor-frontend
cd shared && git pull origin main && cd ..

# 1. shared/contracts/api.md §CTRL 동기화 확인 (응답 형식)
# 2. shared/orchestrator/design_20260526_ctrl_sync.md §Test review 시나리오 확인
# 3. 시나리오 텍스트로 commit draft + before 스크린샷
# 4. 구현
# 5. agent-browser after 스크린샷 (성공/에러 둘 다)
# 6. 회귀 둘러보기
# 7. SendMessage to team-lead 보고 + git commit [T6-FE-CTRL]
# 8. idle 대기
```

## 경계

- 워크트리 `/home/multion/production/pcbang-sensor-frontend` 안에서만
- backend / edge / project root 파일 절대 수정 X
- shared push 절대 금지 — 읽기만
- jest.config·*.test.tsx 신규 X (F-TEST-INFRA는 별도)
- 매장 상세 페이지의 다른 탭/섹션 시각 변경 금지 (이번은 동기화 버튼 한정)
- DESIGN.md 의 색상·폰트 토큰 그대로 사용 — 새 토큰 추가 X
