# [Backend → Edge] MQTT 브로커 변경: Mosquitto → EMQX

날짜: 2026-04-08
발신: Backend 팀
수신: Edge 팀

---

## 변경 사항

MQTT 브로커를 **Mosquitto에서 EMQX Community 5.8.9로 교체**했습니다.
PostgreSQL `mqtt_credentials` 테이블을 EMQX가 직접 실시간 조회하여 인증합니다.

**Edge 코드 변경은 없습니다.** 브로커가 바뀌어도 MQTT 프로토콜은 동일합니다.

---

## Edge 입장에서 달라지는 것

### 없음

- paho-mqtt 연결 코드: 변경 없음
- 인증 방식 (username/password): 변경 없음
- 포트 (8883): 변경 없음
- 토픽 구조: 변경 없음
- QoS, retain, LWT: 변경 없음

### 확인 사항

등록 응답에서 받은 `mqtt_host`, `mqtt_port`, `username`, `password`를 그대로 사용하면 됩니다.

```python
# 기존 코드 그대로 동작
client.username_pw_set(username=data["username"], password=data["password"])
client.connect(host=data["mqtt_host"], port=data["mqtt_port"], keepalive=60)
```

---

## 참고: 인증 흐름

```
Edge 등록 (POST /api/v1/edge/register)
  └─ Backend: mqtt_credentials 테이블에 bcrypt 해시 저장

Edge MQTT 접속 (port 8883)
  └─ EMQX: SELECT password_hash FROM mqtt_credentials
            WHERE username = ? AND is_active = 1
            → bcrypt 검증 → 성공/실패 즉시 반영
```

별도 동기화나 브로커 재시작 없이 등록 즉시 접속 가능합니다.
