# [Backend] 운영 환경 동일 셋업 — QA 모드 전환

_from: Orchestrator | 2026-04-09_

---

## 배경

실서버 배포 테스트 중 다수의 오류가 발견됐다. 현재 개발서버(192.168.0.19)가 실서버와 **동일한 네트워크 조건**으로 셋업 완료되었으므로, 여기서 모든 검증을 마친 뒤 실서버(192.168.0.70)로 이전한다.

---

## 현재 네트워크 구성 (개발서버 기준)

| 서비스 | 외부 주소 | 내부 대상 |
|--------|-----------|-----------|
| Backend API | `https://pcbang-iot-api.multion.synology.me` | `192.168.0.19:8080` |
| Frontend | `https://pcbang-iot.multion.synology.me` | `192.168.0.19:3000` |
| MQTT Broker | `112.220.103.108:8883` | `192.168.0.19:8883` (포트포워딩) |

- Backend/Frontend: Synology 리버스 프록시 → **HTTPS**
- MQTT: TCP 포트포워딩 (HTTP 불가) → 공인 IP 직접 노출

---

## Backend가 확인/수정해야 할 항목

### 1. 환경변수 (`.env`) 검토

운영 환경에서 필요한 환경변수가 모두 설정되어 있어야 한다.

```env
# CORS — 프론트 도메인 반드시 포함
CORS_ORIGINS=http://localhost:3000,https://pcbang-iot.multion.synology.me

# 쿠키 — cross-origin(서로 다른 서브도메인) 대응
COOKIE_DOMAIN=.multion.synology.me

# Edge가 외부에서 접속할 MQTT 주소
EDGE_MQTT_HOST=112.220.103.108
EDGE_MQTT_PORT=8883

# Edge 등록 시크릿 (Edge const.py의 COMPONENT_SECRET_KEY와 동일값)
EDGE_REGISTER_SECRET=<설정된 값>
```

### 2. Cross-Origin 쿠키 설정 확인 (`auth.py`)

```python
# response.set_cookie(...)
samesite="none"
secure=True
domain=settings.COOKIE_DOMAIN  # ".multion.synology.me"
```

### 3. EMQX 설정 확인 (`emqx.conf`)

- `listeners.tcp.default` 포트: **1883** (내부 백엔드 연결용)
- `listeners.ssl.default`: **비활성화** (TLS 미사용)
- 백엔드 MQTT 클라이언트는 내부 `1883`으로 연결
- Edge는 외부 `8883`을 통해 접속 (포트포워딩)

### 4. docker-compose healthcheck 확인

EMQX가 완전히 기동된 후 백엔드가 시작되어야 함.

```yaml
depends_on:
  emqx:
    condition: service_healthy
```

### 5. EMQX 내부 포트 혼선 확인

기존 오류: EMQX 기본 리스너 `tcp:default`가 1883을 이미 점유 → 커스텀 리스너와 충돌.
`listeners.tcp.default`로 **기본 리스너를 직접 재정의**하는 방식인지 확인.

---

## 모드 전환: 개발 → QA/유지보수

이번 스프린트부터 신규 기능 개발은 없다. **통합 테스트에서 발견되는 버그를 즉시 수정**하는 것이 목표다.

### 테스트 시나리오 (전체 팀 공통)

1. DB 초기화 → 아무 데이터 없는 상태에서 시작
2. 관리자 로그인
3. 매장 등록
4. Edge 서버 기동 (외부망에서)
5. Edge → Backend 등록 요청 (MQTT + HTTP)
6. 센서 데이터 수신 확인
7. 알림 발생 확인
8. Frontend에서 전체 흐름 확인

---

## 참고

- 실서버 배포 중 발견된 오류 전체 목록 → `shared/docs/while_publish.md`
- 실서버 이전은 개발서버에서 **모든 시나리오 통과 후** 진행
