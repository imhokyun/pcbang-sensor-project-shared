# Frontend 작업 지시 (Rev.2)

**발신**: Orchestrator | **날짜**: 2026-04-08 | **브랜치**: dev/frontend  
**전체 그림**: `shared/new_request_20260408_rev2.md` 먼저 읽어볼 것

> **Rev.1 대비 변경**: A1 `entity_name` → `custom_name` 확정

---

## 담당 작업 요약

| ID | 작업 | 의존성 | 변경 여부 |
|----|------|--------|---------|
| A3 | 알림음 — 중요도 4·5 강조 소리 | 없음 | 동일 |
| A1 | 알림 팝업 UI — 센서명·매장명·시간 강조, 카메라 작게 | 없음 | **custom_name 사용 확정** |
| A2 | 알림 목록 — 기기 타입별 섹션 분리 | 없음 | 동일 |
| A4-FE | 장시간 개방 알림 표시 확인 | Backend A4-BE + Edge A4-Edge 완료 후 | 동일 |
| B1-FE | 매장 검색 UI + 페이지네이션 | Backend B1-BE 완료 후 (UI 골격 선행 가능) | 동일 |

---

## A3 — 알림음 중요도 차별화

**파일**: `frontend/hooks/useAlertSound.ts`, `frontend/app/page.tsx`

### 현재 상태

`useAlertSound.ts`: `play()` — 단일 비프음.  
`page.tsx`: `playSound()` 호출 (importance 전달 안 함).

### 변경 사항

**`hooks/useAlertSound.ts`**:

```typescript
// play() → play(importance: number) 시그니처 변경
const play = useCallback((importance: number) => {
  if (!enabled) return;
  try {
    const ctx = getCtx();
    const resume = () => {
      if (importance >= 4) {
        playUrgentBeep(ctx);   // 신규: 고주파 + 3회 반복
      } else {
        playBeep(ctx);          // 기존 유지
      }
    };
    ctx.state === "suspended" ? ctx.resume().then(resume) : resume();
  } catch {
    // AudioContext 미지원 환경 무시
  }
}, [enabled, getCtx]);

// 신규 함수 추가 (파일 상단 모듈 레벨)
function playUrgentBeep(ctx: AudioContext) {
  // 880Hz, 200ms 간격으로 3회 반복
  [0, 0.25, 0.5].forEach((offset) => {
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    osc.connect(gain);
    gain.connect(ctx.destination);
    osc.type = "sine";
    osc.frequency.value = 880;
    gain.gain.setValueAtTime(0.3, ctx.currentTime + offset);
    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + offset + 0.2);
    osc.start(ctx.currentTime + offset);
    osc.stop(ctx.currentTime + offset + 0.2);
  });
}
```

**`app/page.tsx`** — `playSound` 호출부 수정:

```typescript
// 기존
if (newAlerts.some((a) => a.is_in_schedule)) {
  playSound();
}

// 변경
if (newAlerts.some((a) => a.is_in_schedule)) {
  const maxImportance = Math.max(...newAlerts.filter(a => a.is_in_schedule).map(a => a.importance));
  playSound(maxImportance);
}
```

---

## A1 — 알림 팝업 UI 개선

**파일**: `frontend/components/VideoPopup.tsx`

### 확정 사항

`entity_name` 필드는 사용하지 않습니다. **`custom_name`** 을 센서 표시명으로 사용합니다.  
(Alert 타입 기준: `alert.custom_name`)

### 레이아웃 변경

현재: 헤더에 `store_name · custom_name · state_from→state_to` 한 줄 + 시각  
변경: 정보를 **시각적으로 강조**하고, 카메라를 더 작게

```
┌─────────────────────────────────────────────────────┐
│  강남점                            중요도: ★★★★☆   │  ← store_name 크게 (18px bold)
│  냉장고1 센서  ·  장시간개방                          │  ← custom_name + type_name (14px)
│  14:32:05   off → on                       [✕]      │  ← 시각 모노스페이스 + state
├─────────────────────────────────────────────────────┤
│  CH1 | CH2 | ...  (카메라 탭, 2개 이상 시)           │
├─────────────────────────────────────────────────────┤
│                                                     │
│    카메라 영상 (전체 높이의 약 45%, 16/9 비율)         │  ← 기존보다 작게
│                                                     │
├─────────────────────────────────────────────────────┤
│  ← 이전   ↑↓ 이전/다음 · Enter 확인 · Esc 닫기   [✓ 확인]  │
└─────────────────────────────────────────────────────┘
```

**헤더 구현 방향**:

```typescript
{/* ── 헤더 ── */}
<div style={{ padding: "16px 20px", borderBottom: "1px solid var(--border-subtle)", background: isHigh ? "rgba(61,14,14,0.4)" : undefined }}>
  {/* 1행: 매장명 + 중요도 */}
  <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "6px" }}>
    <span style={{ fontSize: "18px", fontWeight: 700, color: "var(--text-primary)" }}>
      {alert.store_name}
    </span>
    <div style={{ display: "flex", alignItems: "center", gap: "8px" }}>
      <span style={{ fontSize: "12px", color: "var(--text-dim)" }}>
        {"★".repeat(alert.importance)}{"☆".repeat(5 - alert.importance)}
      </span>
      <button onClick={onClose} style={/* 기존 닫기 버튼 스타일 */}>✕</button>
    </div>
  </div>
  {/* 2행: 센서명 + 타입 */}
  <div style={{ fontSize: "14px", color: "var(--text-muted)", marginBottom: "4px" }}>
    {alert.custom_name}
    {alert.type_name && <span style={{ color: "var(--text-dim)", marginLeft: "8px" }}>· {alert.type_name}</span>}
  </div>
  {/* 3행: 시각 + 상태변화 */}
  <div style={{ display: "flex", alignItems: "center", gap: "12px", fontFamily: "var(--font-mono)", fontSize: "12px", color: "var(--text-dim)" }}>
    <span>{formatTime(alert.timestamp)}</span>
    <span style={{ color: isHigh ? "var(--accent)" : "var(--text-muted)" }}>
      {alert.state_from} → {alert.state_to}
    </span>
    {alerts.length > 1 && <span>{currentIndex + 1} / {alerts.length}</span>}
  </div>
</div>
```

**카메라 영상 영역**: 현재 `aspectRatio: "16/9"` 유지하되 팝업 전체 `maxHeight`를 제한하여 상대적으로 작아 보이도록 조정:

```typescript
// 팝업 컨테이너에 maxHeight 추가
style={{
  width: "100%", maxWidth: "800px",
  maxHeight: "85vh",           // 추가: 화면 높이의 85% 제한
  background: "var(--bg-surface)",
  ...
  overflowY: "auto",           // 추가: 내용 넘칠 때 스크롤
}}
```

---

## A2 — 알림 목록 기기 타입별 섹션 분리

**파일**: `frontend/components/AlertList.tsx`

### 변경 사항

```typescript
import { useMemo } from "react";

// AlertList 컴포넌트 내부
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
- 헤더 "Pending Alerts" 제거 → 각 섹션 헤더로 대체
  - 단, 총 알림 수 배지는 유지 (`alerts.length`)
- 각 섹션: `type_name` 문자열이 섹션 헤더 (예: "냉장고", "출입문", "장시간개방")
- 섹션 내 정렬: `occurred_at` / `timestamp` 내림차순 (최신 먼저)
- 빈 섹션은 렌더링하지 않음

```typescript
{/* 섹션 헤더 예시 */}
<div style={{
  fontSize: "10px", fontWeight: 700, letterSpacing: "0.08em",
  textTransform: "uppercase", color: "var(--text-dim)",
  padding: "8px 0 4px", marginTop: "8px",
  fontFamily: "var(--font-display)",
}}>
  {typeName} <span style={{ color: "var(--accent)", marginLeft: "4px" }}>{alerts.length}</span>
</div>
```

---

## A4-FE — 장시간 개방 알림 표시 확인

Backend·Edge 구현 완료 통보 수신 후:

1. `type_name = '장시간개방'` AlertEvent가 WS로 오면 A2 섹션에 자동 분류됨 — 추가 개발 불필요
2. 실수신 테스트: HA에서 binary_sensor on → threshold 경과 → 알림 화면 확인
3. 결과 `shared/to_orchestrator/from_frontend_20260408_a4_test_result.md` 에 리포트

---

## B1-FE — 매장 검색 UI + 페이지네이션

**파일**: `frontend/app/stores/page.tsx`, `frontend/lib/api.ts`

### 1단계: UI 골격 선행 (Backend 완료 전 가능)

검색창, 페이지네이션 컴포넌트 구조만 구현. API는 Mock/현재 형식 유지.

### 2단계: API 연결 (Backend B1-BE 완료 통보 후)

**`lib/api.ts`** — `storesApi.list` 변경:

```typescript
// 기존
list: () => api.get<Store[]>("/stores"),

// 변경
list: (params?: { q?: string; page?: number; limit?: number }) => {
  const qs = new URLSearchParams();
  if (params) {
    Object.entries(params).forEach(([k, v]) => {
      if (v !== undefined && v !== null && v !== "") qs.set(k, String(v));
    });
  }
  const query = qs.toString();
  return api.get<{ items: Store[]; total: number; page: number; total_pages: number }>(
    `/stores${query ? `?${query}` : ""}`
  );
},
```

**`app/stores/page.tsx`** — 검색 + 페이지네이션 추가:
- 검색 입력창 (300ms debounce, `q` 상태 변경 시 `page=1` 리셋)
- 페이지네이션 UI: `← 이전` / `1 / N 페이지` / `다음 →`
- 로딩 중 Skeleton 유지

---

## 작업 순서 (권장)

```
1. A3 (useAlertSound.ts + page.tsx) — 독립, 먼저 처리
2. A1 (VideoPopup.tsx) — 독립 컴포넌트
3. A2 (AlertList.tsx) — 독립 컴포넌트
4. B1-FE 1단계 (UI 골격) — Backend 무관
5. B1-FE 2단계 (API 연결) — Backend 완료 통보 후
6. A4-FE (테스트) — Backend + Edge 완료 후
```
