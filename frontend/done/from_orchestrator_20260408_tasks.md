# Frontend 작업 지시

**발신**: Orchestrator | **날짜**: 2026-04-08 | **브랜치**: dev/frontend  
**전체 그림**: `shared/new_request_20260408.md` 먼저 읽어볼 것

---

## 담당 작업 요약

| ID | 작업 | 의존성 |
|----|------|--------|
| A3 | 알림음 — 중요도 4·5 강조 소리 | 없음 |
| A1 | 알림 팝업 UI — 매장명·센서명·시간 강조, 카메라 작게 | 없음 |
| A2 | 알림 목록 — 기기 타입별 섹션 분리 | 없음 |
| A4-FE | 장시간 개방 알림 표시 확인 | Backend A4-BE 완료 후 |
| B1-FE | 매장 검색 UI + 페이지네이션 | Backend B1-BE 완료 후 (골격 선행 가능) |

---

## A3 — 알림음 중요도 차별화

**파일**: `hooks/useAlertSound.ts`, `app/page.tsx`

### 현재 상태
`play()` 가 단일 beep 재생. 중요도 무관.

### 변경 사항

**`hooks/useAlertSound.ts`**
```typescript
// play() → play(importance: number) 로 시그니처 변경
const play = useCallback((importance: number) => {
  const ctx = getCtx();
  if (!ctx) return;
  const resume = () => {
    if (importance >= 4) {
      playUrgentBeep(ctx);   // 신규: 고주파 + 3회 반복
    } else {
      playBeep(ctx);          // 기존 유지
    }
  };
  ctx.state === "suspended" ? ctx.resume().then(resume) : resume();
}, [enabled, getCtx]);

// 신규 함수 추가
function playUrgentBeep(ctx: AudioContext) {
  // 880Hz, 200ms 간격으로 3회 반복
  const times = [0, 0.25, 0.5];
  times.forEach((offset) => {
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    osc.connect(gain);
    gain.connect(ctx.destination);
    osc.frequency.value = 880;
    gain.gain.setValueAtTime(0.3, ctx.currentTime + offset);
    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + offset + 0.2);
    osc.start(ctx.currentTime + offset);
    osc.stop(ctx.currentTime + offset + 0.2);
  });
}
```

**`app/page.tsx`** — play 호출부에 importance 전달
```typescript
playSound(newAlert.importance);  // 기존: playSound()
```

---

## A1 — 알림 팝업 UI 개선

**파일**: `components/VideoPopup.tsx`

### 변경 사항

팝업 레이아웃을 **정보 우선**으로 재구성:

```
┌─────────────────────────────────────┐
│  [매장명]  강남점                    │  ← 크게 (18~20px, bold)
│  [센서]   냉장고1 센서               │  ← 중간 (14px)
│  [시각]   14:32:05    ★★★★☆        │  ← 모노스페이스 + 중요도
├─────────────────────────────────────┤
│                                     │
│      카메라 화면 (전체의 45%)        │  ← 기존보다 작게
│                                     │
├─────────────────────────────────────┤
│  [확인]           [◀ 이전] [다음 ▶] │
└─────────────────────────────────────┘
```

현재 Alert 타입에서 사용할 필드:
```typescript
alert.store_name    // 매장명
alert.entity_name   // 센서명 (없으면 alert.type_name 사용)
alert.type_name     // 기기 타입
alert.occurred_at   // 발생 시각
alert.importance    // 중요도 (★ 아이콘으로 표시)
```

---

## A2 — 알림 목록 기기 타입별 섹션 분리

**파일**: `components/AlertList.tsx`

### 변경 사항

```typescript
// 기존: alerts 배열 그대로 렌더링
// 변경: type_name 기준 groupBy 후 섹션별 렌더링

const grouped = useMemo(() => {
  const map = new Map<string, Alert[]>();
  for (const alert of alerts) {
    const key = alert.type_name ?? "기타";
    if (!map.has(key)) map.set(key, []);
    map.get(key)!.push(alert);
  }
  return map;
}, [alerts]);
```

렌더링:
- 각 섹션 헤더: `type_name` 문자열 (예: "냉장고", "출입문", "금고")
- 빈 섹션 숨기기
- 섹션 내 정렬: `occurred_at` 내림차순
- `type_name == null` → "기타" 섹션
- 헤더 "PENDING ALERTS" 제거 (각 섹션 헤더로 대체)

---

## A4-FE — 장시간 개방 알림 표시

Backend가 `type_name = '장시간개방'` AlertEvent를 WebSocket으로 내려줍니다.

**확인 작업**: 기존 알림 표시 로직이 그대로 동작하는지 체크.  
A2 작업에서 type_name으로 섹션을 분리하므로, "장시간개방" 섹션이 자동으로 생깁니다.  
별도 개발 없이 동작 예상 — Backend 완료 후 실제 알림 수신 테스트만 진행해주세요.

---

## B1-FE — 매장 검색 UI + 페이지네이션

**파일**: `app/stores/page.tsx`, `lib/api.ts`

### lib/api.ts 변경
```typescript
// 기존
list: () => api.get<Store[]>("/stores"),

// 변경
list: (params?: { q?: string; page?: number; limit?: number }) =>
  api.get<{ items: Store[]; total: number; page: number; total_pages: number }>(
    "/stores",
    { params }
  ),
```

### app/stores/page.tsx 변경
- 검색 입력창 추가 (300ms debounce)
- 검색어 변경 시 page=1로 리셋
- 페이지네이션 UI: 이전/다음 버튼 + "1 / N 페이지" 표시
- 로딩 중 skeleton 유지

**Backend API 완료 전 선행 가능**: UI 골격 + 타입 변경은 미리 구현해두면 됩니다.  
Backend 완료 통보는 `shared/to_frontend/` 에 별도 파일로 올라옵니다.

---

## 작업 순서

```
1. A3 (useAlertSound 훅) — 다른 파일에 영향 최소화, 먼저 처리
2. A1 (VideoPopup 레이아웃) — 독립 컴포넌트
3. A2 (AlertList 섹션 분리) — 독립 컴포넌트
4. B1-FE (매장 검색 UI) — Backend 완료 전이라도 UI 골격 선행
5. A4-FE (장시간개방 알림 테스트) — Backend 완료 후
```
