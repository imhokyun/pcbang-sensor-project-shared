# 운영 배포 중 변경/발견 사항 (2026-04-09)

서버: Ubuntu 24.04 / 192.168.0.70
외부 접속: Synology 리버스 프록시

---

## 외부 접속 구성

| 서비스 | 외부 주소 | 내부 대상 |
|--------|-----------|-----------|
| Frontend | `https://pcbang-iot.multion.synology.me` | `192.168.0.70:3000` |
| Backend API | `https://pcbang-iot-api.multion.synology.me` | `192.168.0.70:8080` |
| MQTT | `112.220.103.108:8883` | `192.168.0.70:8883` (포트포워딩) |

- MQTT는 TCP 프로토콜이라 Synology 리버스 프록시(HTTP 전용) 사용 불가 → 라우터 포트포워딩으로 처리
- Cloudflare Tunnel도 TCP 미지원으로 사용하지 않음

---

## EMQX 설정 수정

### 포트 충돌 해결
- EMQX 5.x 기본 `tcp:default` 리스너가 1883을 점유 → 커스텀 `tcp:backend`와 충돌
- 해결: `listeners.tcp.default`로 재정의 (기본 리스너를 덮어씀)
- `listeners.ssl.default`도 8883 점유 → `enable = false`로 비활성화

### docker-compose healthcheck 추가
- EMQX가 완전히 기동된 후 백엔드가 시작되도록 healthcheck 추가
- `emqx ctl status | grep -q 'is started'`
- 백엔드 `depends_on` 조건을 `service_started` → `service_healthy`로 변경

---

## Cross-Origin 쿠키 문제

### 증상
- 프론트(`pcbang-iot.multion.synology.me`)에서 백엔드(`pcbang-iot-api.multion.synology.me`)로 로그인 요청 → 성공하지만 쿠키가 브라우저에 저장 안 됨
- 로그인 후 `/`로 이동하면 middleware가 session_id 쿠키를 못 찾고 다시 `/login`으로 리다이렉트

### 원인
1. **Mixed Content**: 프론트가 HTTPS인데 API URL이 HTTP로 하드코딩 → 브라우저 차단
2. **SameSite=Lax**: cross-origin 요청 시 쿠키 전송 안 됨
3. **Cookie Domain**: 서로 다른 서브도메인이라 쿠키가 공유 안 됨

### 해결
1. Frontend `api.ts`/`providers.tsx`: `window.location.hostname` 기반 하드코딩 제거 → `NEXT_PUBLIC_API_URL`/`NEXT_PUBLIC_WS_URL` 환경변수 사용
2. Backend `auth.py`: `SameSite=None`, `Secure=True` 설정
3. Backend `config.py`: `COOKIE_DOMAIN` 환경변수 추가 (`.env`에서 `.multion.synology.me` 설정)
4. Backend CORS: `https://pcbang-iot.multion.synology.me` 추가

---

## Frontend API 응답 형식 불일치

### 증상
- `/stores`, `/logs` 페이지에서 `Cannot read properties of undefined (reading 'map')` / `(reading 'length')`

### 원인
- `storesApi.list()`가 `res.items`를 기대하지만, 백엔드는 `{"success": true, "data": [...]}`로 배열을 직접 반환
- `api.ts`의 wrapper가 `data`를 추출하므로 `res`는 배열이고 `res.items`는 `undefined`

### 해결
- `stores/page.tsx`, `logs/page.tsx`: `Array.isArray(res) ? res : res.items ?? []`로 방어 코드 추가

---

## 환경변수 정리

### Backend `.env` 추가 항목
```
CORS_ORIGINS=http://localhost:3000,https://pcbang-iot.multion.synology.me
COOKIE_DOMAIN=.multion.synology.me
EDGE_MQTT_HOST=112.220.103.108
```

### Frontend `.env`
```
NEXT_PUBLIC_API_URL=https://pcbang-iot-api.multion.synology.me
NEXT_PUBLIC_WS_URL=wss://pcbang-iot-api.multion.synology.me/ws
```

---

## Next.js 버전 이슈

- Next.js 16.2.2 (Turbopack) production build에서 chunk 누락 버그 발견
  - 빌드된 HTML이 존재하지 않는 chunk(`0l8w99q_729ja.js`)를 참조 → 500 에러
  - Next.js 15 다운그레이드로 해결 가능하나, ESLint 빌드 에러 발생
  - 최종: Next.js 16 유지 + `next.config.ts`에서 `eslint.ignoreDuringBuilds` 불필요 (16에서 미지원)
  - **주의**: `npm run dev`로 띄운 프로세스가 nohup으로 남아있으면 포트를 점유하므로 `fuser -k 3000/tcp`로 확실히 정리 후 PM2 시작

---

## 프로세스 관리

| 서비스 | 관리 방식 | 자동 시작 |
|--------|----------|-----------|
| PostgreSQL | Docker (`restart: unless-stopped`) | 서버 재부팅 시 자동 |
| EMQX | Docker (`restart: unless-stopped`) | 서버 재부팅 시 자동 |
| Backend | Docker (`restart: unless-stopped`) | 서버 재부팅 시 자동 |
| Frontend | PM2 (`pm2 startup` + `pm2 save`) | 서버 재부팅 시 자동 |

### PM2 명령어
```bash
pm2 start npm --name "pcbang-frontend" --cwd /path/to/frontend -- run start
pm2 save          # 현재 상태 저장
pm2 startup       # 부팅 시 자동 시작 설정
pm2 logs           # 로그 확인
pm2 restart pcbang-frontend  # 재시작
```

---

## Edge 시크릿 키

- `EDGE_REGISTER_SECRET` (Backend `.env`)과 `COMPONENT_SECRET_KEY` (Edge `const.py`)를 동일한 난수값으로 설정 완료
- Edge 등록 시 이 값으로 인증

---

## 로그인 계정

- admin1 ~ admin5 / 비밀번호: `.env`의 `ADMIN_PASSWORD` 값
- seed 마이그레이션은 `admin1234`로 하드코딩되어 있으므로, 비밀번호 변경 시 DB 직접 UPDATE 필요
