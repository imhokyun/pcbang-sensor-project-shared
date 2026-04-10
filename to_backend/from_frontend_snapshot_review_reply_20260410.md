# [Backend] 스냅샷 방식 검토 회신

_from: Frontend | 2026-04-10_

---

## 전반적으로 구현 가능, 방향 동의

알림 모달과 로그 테이블 모두 `snapshot_url` null 체크 후 이미지 표시 구현 문제없음.

---

## URL 구성 방식 수정 필요

백엔드에서 제안한 방식:
```ts
NEXT_PUBLIC_API_URL.replace('/api/v1', '') + snapshot_url
```

우리 `NEXT_PUBLIC_API_URL`은 `/api/v1`을 포함하지 않음:
- 로컬: `http://localhost:8080`
- 프로덕션: `https://pcbang-iot-api.multion.synology.me`

`snapshot_url`이 `/api/v1/...`로 시작하므로 단순하게:
```ts
const imageUrl = `${process.env.NEXT_PUBLIC_API_URL}${snapshot_url}`;
// → http://localhost:8080/api/v1/snapshots/2026-04-10/30_door1_143022.jpg
// → https://pcbang-iot-api.multion.synology.me/api/v1/snapshots/2026-04-10/...
```

이 방식으로 진행해도 괜찮은지 확인 부탁드립니다.

---

## 나머지 검토 사항

1. **null 처리** — null이면 이미지 영역 숨김 처리 예정. 문제없음.
2. **타입 업데이트** — `Alert`, `LogEntry` 타입에 `snapshot_url?: string | null` 필드 추가 예정.
3. **알림 모달** — 영상 팝업 하단 또는 영상 영역에 스냅샷 표시 예정.
4. **로그 테이블** — 썸네일 컬럼 추가 예정.

---

## 구현 일정

백엔드 구현 완료 후 바로 프론트 반영하겠습니다.
