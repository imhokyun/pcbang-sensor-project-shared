# [Backend → Edge] MQTT 포트 변경: 1883 → 8883 (필수 반영)

날짜: 2026-04-08  
발신: Backend 팀  
수신: Edge 팀

---

## 변경 내용

Mosquitto에 **포트 기반 접근 분리 + go-auth 인증**을 적용했습니다.

| 포트 | 용도 | 인증 |
|------|------|------|
| 1883 | Backend 전용 (localhost 내부) | 없음 (외부 접근 차단) |
| **8883** | **Edge 전용** | **username/password 필수** |

**Edge는 반드시 8883으로 연결해야 합니다.**  
1883은 서버 내부(localhost)에서만 접근 가능하므로 외부에서 연결 시 거부됩니다.

---

## Edge 코드 수정 필요 사항

### 1. 포트 하드코딩 제거

기존에 1883을 하드코딩했다면 제거하세요. 등록 응답에서 받은 `mqtt_port`를 사용해야 합니다.

```python
# 등록 응답에서 수신
mqtt_host = data["mqtt_host"]  # 서버 IP
mqtt_port = data["mqtt_port"]  # 8883 (항상)
username  = data["username"]   # str(store_id)
password  = data["password"]   # 랜덤 생성된 MQTT 비밀번호
```

### 2. TLS 설정 제거

8883이지만 **TLS(SSL)는 사용하지 않습니다.** 이전 계약에 "TLS" 오기재가 있었습니다.  
plain MQTT + username/password 인증입니다.

```python
# ✅ 올바른 연결
client = mqtt.Client(client_id=f"pcbang-edge-{store_id}", clean_session=False)
client.username_pw_set(username, password)
# tls_set() 호출 없음

client.connect(mqtt_host, mqtt_port, keepalive=60)
```

### 3. paho-mqtt 연결 파라미터 확인

```python
client.username_pw_set(username=str(store_id), password=received_password)
client.connect(host=mqtt_host, port=8883, keepalive=60)
```

---

## 로컬 개발 환경

개발 환경에서도 동일하게 8883을 사용합니다.  
`docker compose up -d db mosquitto` 실행 후 Edge 등록 → 8883 연결.

등록 엔드포인트: `POST http://localhost:8080/api/v1/edge/register`  
응답의 `mqtt_port`가 8883으로 옵니다.

---

## 확인 요청

- [ ] paho-mqtt 연결 시 포트 8883 사용 확인
- [ ] TLS(tls_set) 호출 없는지 확인  
- [ ] 등록 응답의 `mqtt_port` 값을 그대로 사용하는지 확인 (하드코딩 금지)
