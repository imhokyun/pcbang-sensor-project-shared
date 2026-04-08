# [Backend → Edge] EMQX 구동 완료 — MQTT 접속 방법 변경 필수

날짜: 2026-04-08  
발신: Backend 팀  
수신: Edge 팀  

---

## 요약

EMQX가 정상 구동되었습니다. **Mosquitto는 완전히 제거**되었으며, 지금부터 Edge는 EMQX로 접속해야 합니다.  
접속 방법이 바뀌었으니 아래 내용을 반드시 반영해 주세요.

---

## 변경된 접속 방법

| 항목 | 이전 (Mosquitto) | 현재 (EMQX) |
|------|-----------------|-------------|
| 포트 | 1883 | **8883** |
| 인증 | 없음 (anonymous) | **username + password 필수** |
| username | - | `store_id` (예: `"30584"`) |
| password | - | `/edge/register` API 응답값 |

---

## 적용 절차

### 1. `/edge/register` 재호출 (새 비밀번호 발급)

```
POST http://{서버IP}:8000/api/v1/edge/register
Content-Type: application/json

{
  "store_id": 30584,
  "secret_key": "change-me-shared-with-edge-component"
}
```

응답 예시:
```json
{
  "success": true,
  "data": {
    "mqtt_host": "서버IP",
    "mqtt_port": 8883,
    "username": "30584",
    "password": "발급된_평문_비밀번호"
  }
}
```

### 2. MQTT 접속 코드 수정

```python
# 응답에서 받은 값으로 그대로 연결
client.username_pw_set(username=data["username"], password=data["password"])
client.connect(host=data["mqtt_host"], port=data["mqtt_port"], keepalive=60)
```

- 토픽 구조: 변경 없음
- QoS, retain, LWT: 변경 없음
- paho-mqtt 코드: username_pw_set + 포트만 수정

---

## 인증 흐름

```
1. Edge → POST /edge/register → Backend → DB에 bcrypt 해시 저장
2. Edge → MQTT CONNECT (port 8883, username, password)
        → EMQX가 DB 직접 조회 → bcrypt 검증 → 즉시 허용/거부
```

재등록 즉시 EMQX가 반영합니다. 브로커 재시작 불필요.

---

## 현재 서버 상태

- EMQX 5.8.9: 정상 구동 ✅
- PostgreSQL 인증: 정상 ✅  
- Backend API: 정상 구동 ✅ (port 8000)
- paho-mqtt로 실 접속 인증 테스트 완료 ✅
