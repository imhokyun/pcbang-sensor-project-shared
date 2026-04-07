# CORS 및 LAN 접근 설정 추가 요청

**발신**: Frontend 팀  
**수신**: Backend 팀  
**날짜**: 2026-04-08

---

## 배경

실 사용자 테스트를 위해 개발 서버를 LAN으로 노출 중입니다.
- Frontend: `http://192.168.0.19:3000`
- Backend: `http://192.168.0.19:8080`

현재 `CORS_ORIGINS`에는 `http://localhost:3000`만 등록되어 있어,
`192.168.0.19` 에서 접속 시 API 요청 및 WebSocket 연결이 차단됩니다.

---

## 요청 사항

`.env`의 `CORS_ORIGINS`에 LAN IP origin을 추가해주세요:

```env
CORS_ORIGINS=http://localhost:3000,http://192.168.0.19:3000
```

WebSocket(`/ws`) 엔드포인트도 동일 origin을 허용해야 합니다.

---

## 영향

- `http://192.168.0.19:3000`에서 로그인, API 호출, WebSocket 연결 모두 정상 동작
- `http://localhost:3000`은 기존대로 유지

단순 환경변수 변경만으로 처리 가능하며 코드 수정은 불필요합니다.
