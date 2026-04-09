# [Edge] 운영 환경 동일 셋업 — QA 모드 전환

_from: Orchestrator | 2026-04-09_

---

## 배경

실서버 배포 테스트 중 다수의 오류가 발견됐다. 현재 개발서버(192.168.0.19)가 실서버와 **동일한 네트워크 조건**으로 셋업 완료되었으므로, 여기서 모든 검증을 마친 뒤 실서버(192.168.0.70)로 이전한다.

---

## 현재 네트워크 구성

| 서비스 | 외부 주소 | 내부 대상 |
|--------|-----------|-----------|
| Backend API | `https://pcbang-iot-api.multion.synology.me` | `192.168.0.19:8080` |
| MQTT Broker | `112.220.103.108:8883` | `192.168.0.19:8883` (포트포워딩) |

> **Edge는 완전히 다른 외부망에서 접속한다.**  
> 내부 IP(192.168.0.x)는 Edge에서 절대 사용 불가. 반드시 외부 주소를 사용해야 한다.

---

## Edge가 확인/수정해야 할 항목

### 1. `const.py` 연결 주소

```python
# Backend 등록 API
BACKEND_API_URL = "https://pcbang-iot-api.multion.synology.me"

# MQTT Broker (외부 공인 IP)
MQTT_HOST = "112.220.103.108"
MQTT_PORT = 8883

# 등록 시크릿 (Backend .env의 EDGE_REGISTER_SECRET과 동일값)
COMPONENT_SECRET_KEY = "<설정된 값>"
```

### 2. MQTT 연결 방식 확인

- 포트 **8883** (TLS 없음, plain TCP)
- username / password 인증 방식 (EMQX에 등록된 자격증명)
- TLS 설정이 있으면 **제거** (현재 8883은 plain TCP 포트포워딩)

### 3. Backend 등록 요청 흐름 확인

Edge 최초 기동 시:
1. `POST https://pcbang-iot-api.multion.synology.me/edge/register` 로 등록 요청
2. Backend로부터 `store_id`, MQTT 자격증명 수신
3. MQTT `112.220.103.108:8883`으로 연결
4. 센서 데이터 publish 시작

각 단계가 외부망에서 정상 동작하는지 확인.

### 4. Home Assistant 설정

`const.py`의 주소가 개발 시 내부 IP로 하드코딩되어 있었다면 반드시 외부 주소로 변경.  
HA config_flow에서 사용자가 입력하는 값인지, 코드에 박혀있는 값인지 확인.

---

## 모드 전환: 개발 → QA/유지보수

신규 기능 개발 없음. **외부망에서 Edge를 처음부터 설치·등록하는 전체 흐름을 테스트하며 오류를 찾아 수정**하는 것이 목표.

### 테스트 시나리오 (Edge 담당)

1. 완전히 다른 네트워크(예: LTE 핫스팟 또는 다른 서브넷)에서 Edge 서버 기동
2. HA config_flow에서 Backend URL, 시크릿 입력 → 등록 요청
3. Backend로부터 응답 수신 확인
4. MQTT `112.220.103.108:8883` 연결 확인
5. 센서 데이터 publish → Frontend 알림 수신까지 전체 흐름 확인
6. 릴레이 명령 수신 확인

---

## 참고

- 실서버 배포 중 발견된 오류 전체 목록 → `shared/docs/while_publish.md`
- 실서버 이전은 개발서버에서 **모든 시나리오 통과 후** 진행
