# [Edge → Backend] edge/register 응답의 mqtt_host 수정 요청

**발신**: Edge  
**수신**: Backend  
**일시**: 2026-04-08

---

## 문제

`POST /api/v1/edge/register` 응답의 `mqtt_host`가 `"localhost"`로 내려오고 있습니다.

HA가 Docker 컨테이너 안에서 실행되므로, `localhost`는 HA 컨테이너 자신을 가리킵니다.  
EMQX(host에서 실행 중)에 연결하려면 `host.docker.internal`이 필요합니다.

```
ConnectionRefusedError: [Errno 111] Connection refused
연결 시도: localhost:8883  ← 실패
```

---

## 요청

`/edge/register` 응답의 `mqtt_host`를 다음 중 하나로 수정해 주세요:

- **로컬 개발 환경**: `"host.docker.internal"` (Docker에서 host 접근용)
- **운영 환경 (RP4)**: 실제 서버 외부 IP (예: `"192.168.x.x"`)

응답 예시:
```json
{
  "success": true,
  "data": {
    "mqtt_host": "host.docker.internal",
    "mqtt_port": 8883,
    "username": "30584",
    "password": "..."
  }
}
```

---

Edge 코드 변경은 없습니다. 응답값을 그대로 사용하는 구조입니다.
