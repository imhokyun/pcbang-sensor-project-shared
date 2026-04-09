# [Backend] go2rtc HTTP + WebSocket 프록시 엔드포인트 요청

_from: Frontend | 2026-04-10_

---

## 배경

프론트가 HTTPS(`https://pcbang-iot.multion.synology.me`)로 서빙되면서 외부 매장 Edge 기기의 go2rtc(`http://192.168.x.x:1984`)로 직접 접근이 차단됨.

**브라우저 에러:**
```
Mixed Content: iframe src 'http://192.168.20.30:1984/stream.html' blocked
Mixed Content: WebSocket 'ws://192.168.20.30:1984/api/ws?src=ch1_main' blocked
```

프론트에서는 WebSocket 프록시를 구현할 수 없음 → 백엔드(서버)에서 처리 필요.

---

## 요청 엔드포인트

백엔드가 go2rtc까지 내부망으로 직접 접근 가능하므로 프록시 역할을 해달라는 요청.

### 1. HTTP 프록시 (stream.html + JS 파일)

```
GET /go2rtc/{host}/{path:path}
```

동작:
- `http://{host}:1984/{path}` 로 요청을 전달
- 응답이 HTML이면, 내부 WebSocket URL을 프록시 경로로 치환

HTML 내 치환 예시:
```
ws://{host}:1984/api/ws  →  wss://pcbang-iot-api.multion.synology.me/go2rtc/{host}/api/ws
```

예시 요청:
```
GET /go2rtc/192.168.20.30/stream.html?src=ch1_main
→ http://192.168.20.30:1984/stream.html?src=ch1_main 프록시
```

### 2. WebSocket 프록시

```
WebSocket /go2rtc/{host}/api/ws?src={src}
```

동작:
- `ws://{host}:1984/api/ws?src={src}` 로 WebSocket 브릿지

---

## 프론트 변경 예정

백엔드 프록시 완성 시, 프론트에서 `stream_url` 변환 로직 추가:

```ts
// http://192.168.20.30:1984/stream.html?src=ch1_main
// → https://pcbang-iot-api.multion.synology.me/go2rtc/192.168.20.30/stream.html?src=ch1_main

function toProxiedStreamUrl(streamUrl: string): string {
  const url = new URL(streamUrl);
  const apiBase = process.env.NEXT_PUBLIC_API_URL ?? "";
  return `${apiBase}/go2rtc/${url.hostname}${url.pathname}${url.search}`;
}
```

- DB의 `stream_url`은 현재 형식 유지 (`http://IP:1984/...`)
- 프론트에서 HTTPS 환경일 때만 프록시 URL로 변환
- go2rtc 포트는 1984 고정으로 가정

---

## 참고

- go2rtc 포트: 1984 (Edge CLAUDE.md 기준 고정값)
- WS URL은 `video-rtc.js`가 `stream.html` 내 `<video-stream src="...">` 속성 기반으로 구성
- 프록시된 HTML의 `src` 속성이 `wss://api-server/go2rtc/{host}/api/ws?...` 로 되어야 브라우저 차단 없음
