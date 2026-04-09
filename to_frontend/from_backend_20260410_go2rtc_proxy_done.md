# [Frontend] go2rtc 프록시 구현 완료

_from: Backend | 2026-04-10_

---

## 구현된 엔드포인트

### HTTP 프록시
```
GET /api/v1/go2rtc/{host}/{path:path}
```
- `http://{host}:1984/{path}` 로 요청 전달
- HTML 응답의 경우 내부 `ws://{host}:1984/api/ws` URL을 자동으로 아래로 치환:
  ```
  wss://pcbang-iot-api.multion.synology.me/api/v1/go2rtc/{host}/api/ws
  ```

### WebSocket 프록시
```
WS /api/v1/go2rtc/{host}/api/ws?src={src}
```
- `ws://{host}:1984/api/ws?src={src}` 로 브릿지

---

## 프론트 사용 예시

```ts
// stream_url: "http://192.168.20.30:1984/stream.html?src=ch1_main"
// → 프록시 URL:
"https://pcbang-iot-api.multion.synology.me/api/v1/go2rtc/192.168.20.30/stream.html?src=ch1_main"
```

프론트에서 전달해준 `toProxiedStreamUrl()` 로직 그대로 동작합니다.  
`NEXT_PUBLIC_API_URL` 이 `https://pcbang-iot-api.multion.synology.me/api/v1` 이면 됩니다.

---

## 참고

- go2rtc 포트 1984 고정 사용
- 인증(세션 쿠키) 없이 접근 가능 — stream URL은 이미 매장 상세에서 인증된 사용자만 받음
- Edge 연결 실패 시 502, 타임아웃 시 504 반환
