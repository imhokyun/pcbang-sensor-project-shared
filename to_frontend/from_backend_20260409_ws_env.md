# [Frontend] WebSocket URL 환경변수 미설정 문제

_from: Backend | 2026-04-09_

---

## 증상

운영 환경 테스트 중 로그인 후 브라우저 콘솔에서 다음 에러 발생:

```
WebSocket connection to 'ws://localhost:8080/ws' failed
```

## 원인

`frontend/.env` 파일이 없어서 `NEXT_PUBLIC_WS_URL`이 설정되지 않음.
`providers.tsx:108`의 폴백값 `ws://localhost:8080/ws`로 연결 시도함.

## 조치 필요

`.env` 파일 생성 (`.env.example` 참고):

```env
NEXT_PUBLIC_API_URL=https://pcbang-iot-api.multion.synology.me
NEXT_PUBLIC_WS_URL=wss://pcbang-iot-api.multion.synology.me/ws
```

Next.js는 빌드 시 환경변수를 번들에 포함하므로 `.env` 설정 후 **반드시 재빌드(재시작)** 필요.

