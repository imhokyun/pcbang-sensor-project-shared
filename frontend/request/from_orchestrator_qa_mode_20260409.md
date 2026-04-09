# [Frontend] 운영 환경 동일 셋업 — QA 모드 전환

_from: Orchestrator | 2026-04-09_

---

## 배경

실서버 배포 테스트 중 다수의 오류가 발견됐다. 현재 개발서버(192.168.0.19)가 실서버와 **동일한 네트워크 조건**으로 셋업 완료되었으므로, 여기서 모든 검증을 마친 뒤 실서버(192.168.0.70)로 이전한다.

---

## 현재 네트워크 구성

| 서비스 | 외부 주소 | 내부 대상 |
|--------|-----------|-----------|
| Frontend | `https://pcbang-iot.multion.synology.me` | `192.168.0.19:3000` |
| Backend API | `https://pcbang-iot-api.multion.synology.me` | `192.168.0.19:8080` |
| MQTT Broker | `112.220.103.108:8883` | `192.168.0.19:8883` (포트포워딩) |

- Frontend/Backend 모두 **HTTPS** (Synology 리버스 프록시)
- 프론트와 백엔드는 **서로 다른 서브도메인** → cross-origin 쿠키 이슈 있었음 (while_publish.md 참조)

---

## Frontend가 확인/수정해야 할 항목

### 1. 환경변수 기반 API URL 확인

하드코딩된 IP/포트가 **남아있으면 안 된다**. `api.ts`, `providers.tsx` 등 전체 확인.

```env
# .env.production (서버 배포 시)
NEXT_PUBLIC_API_URL=https://pcbang-iot-api.multion.synology.me
NEXT_PUBLIC_WS_URL=wss://pcbang-iot-api.multion.synology.me/ws
```

### 2. Mixed Content 없음 확인

프론트가 HTTPS로 서빙될 때 HTTP API 호출이 있으면 브라우저가 차단한다.  
`NEXT_PUBLIC_API_URL`이 `https://`인지 확인.

### 3. 쿠키 기반 인증 흐름 확인

- 로그인 성공 후 쿠키가 브라우저에 저장되는지 확인  
- 저장 후 `/`로 리다이렉트 시 middleware가 쿠키를 정상 인식하는지 확인  
- cross-origin이므로 fetch 시 `credentials: 'include'` 필수

### 4. API 응답 형식 방어 코드 확인

배포 중 발견된 버그: `res.items`로 접근하는데 백엔드는 배열을 직접 반환.

```ts
// 수정 패턴
Array.isArray(res) ? res : (res.items ?? [])
```

`stores/page.tsx`, `logs/page.tsx` 등 `.items`를 사용하는 모든 곳 확인.

### 5. PM2 배포 환경에서의 빌드 확인

- `npm run build` 후 `pm2 start`로 구동
- 빌드 전 `fuser -k 3000/tcp`로 포트 정리 필수
- Next.js 16 + `next.config.ts`: `eslint.ignoreDuringBuilds` 옵션 불필요 (16에서 미지원, 제거 확인)

### 6. `next.config.ts` 확인

standalone 빌드 모드(`output: 'standalone'`)가 활성화되어 있어야 PM2에서 `.next/standalone/server.js`로 구동 가능.

---

## 모드 전환: 개발 → QA/유지보수

신규 기능 개발 없음. **통합 테스트 시나리오를 직접 브라우저에서 실행하며 오류를 찾아 수정**하는 것이 목표.

### 테스트 시나리오 (전체 팀 공통)

1. DB 초기화 → 아무 데이터 없는 상태에서 시작
2. `https://pcbang-iot.multion.synology.me` 접속 → `/login`으로 리다이렉트 확인
3. 로그인 → 쿠키 저장 확인 → `/` 이동 확인
4. 매장 등록 → 목록 표시 확인
5. Edge 연동 후 센서 데이터/알림 수신 확인
6. 전체 페이지에서 콘솔 에러 없음 확인

---

## 참고

- 실서버 배포 중 발견된 오류 전체 목록 → `shared/docs/while_publish.md`
- 실서버 이전은 개발서버에서 **모든 시나리오 통과 후** 진행
