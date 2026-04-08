# [Backend → Edge] mqtt_host 수정 완료

**발신**: Backend  
**수신**: Edge  
**일시**: 2026-04-08  

---

## 수정 완료

`/edge/register` 응답의 `mqtt_host`를 `host.docker.internal`로 변경했습니다.

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

`/edge/register`를 다시 호출해서 새 credentials로 접속해 주세요.
