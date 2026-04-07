# 설치 담당자(Installer) 워크플로우

새 매장에 Edge 기기(Raspberry Pi 4)를 설치하고 시스템에 연동하는 절차.

---

## 전제 조건

- 관리자로부터 `store_id` 사전 발급받을 것 (예: `store_001`)
  - 관리자가 대시보드 `/stores` 에서 매장 등록 후 store_id 전달
- RP4에 Home Assistant OS 설치 완료
- RP4가 인터넷 연결 가능한 네트워크에 연결됨
- DVR이 로컬 네트워크에 연결되어 RTSP 스트림 접근 가능

---

## 전체 흐름

```
[사전] 관리자가 대시보드에서 매장 등록 → store_id 발급
    │
    ▼
1. RP4에 HA OS 설치 및 부팅
    │
    ▼
2. HA Custom Component 설치
    │
    ▼
3. Config Flow에서 store_id 입력 → Backend 자동 등록
    │
    ▼
4. go2rtc 설치 및 DVR RTSP 연결 확인
    │
    ▼
5. 관리자가 대시보드에서 카메라/센서/릴레이 설정
    │
    ▼
6. 동작 확인
```

---

## Step 1. Raspberry Pi 4 설치

1. Home Assistant OS 이미지를 SD 카드 또는 SSD에 플래시
2. RP4 부팅 후 `http://homeassistant.local:8123` 접속 가능 확인
3. HA 초기 설정 완료 (계정 생성 등)
4. RP4 고정 IP 설정 권장 (라우터 DHCP 고정 또는 HA 네트워크 설정)

---

## Step 2. HA Custom Component 설치

1. HA 파일 시스템에서 `custom_components/pcbang_sensor/` 폴더에 Component 파일 복사
   - HACS를 통한 설치 또는 수동 파일 복사
2. HA 재시작

---

## Step 3. Component 초기 설정 (Config Flow)

```
HA 대시보드 → 설정 → 통합 → "PCBang Sensor" 추가
    │
    ├─ store_id 입력 (관리자로부터 받은 값, 예: store_001)
    │
    └─ 확인 클릭
           │
           Component → Backend POST /api/v1/edge/register
               body: { "store_id": "store_001", "secret_key": "<하드코딩된 공유 키>" }
               │
               ├─ 성공: MQTT credentials 수신 → Component 내부 저장
               │        { "mqtt_host": "...", "mqtt_port": 8883,
               │          "username": "store_001", "password": "..." }
               │
               └─ 실패: 에러 메시지 표시 → store_id 재확인 후 재시도
```

**등록 성공 후 자동 동작:**
- Mosquitto 연결 (TLS, keepAlive=60)
- LWT 등록 (`pcbang/store_001/status` offline 메시지)
- 즉시 heartbeat publish (`status: "online"`)
- `config/monitored_entities` 구독 (초기값 빈 목록 수신)
- 이후 30초마다 heartbeat 자동 전송

**대시보드에서 확인:**
- `/stores/{store_id}` → 매장 상태가 "online"으로 변경되면 연결 성공

---

## Step 4. go2rtc 설치 및 CCTV 연결

### 4-1. go2rtc 설치

RP4에서 go2rtc 바이너리 실행:

```bash
# go2rtc 다운로드 및 실행
wget https://github.com/AlexxIT/go2rtc/releases/latest/download/go2rtc_linux_arm64
chmod +x go2rtc_linux_arm64
./go2rtc_linux_arm64
```

- 기본 포트: API `1984`, WebRTC `8555`
- HA add-on으로 설치하는 방법도 가능

### 4-2. DVR RTSP 스트림 확인

DVR 설명서에서 RTSP URL 형식 확인 (예시):
```
rtsp://{dvr_ip}:554/channel/01/main   # 메인 스트림
rtsp://{dvr_ip}:554/channel/01/sub    # 서브 스트림
```

VLC 또는 go2rtc 웹 UI (`http://{rp4_ip}:1984`)에서 스트림 접속 확인.

### 4-3. go2rtc URL을 대시보드에 등록

관리자가 `/stores/{store_id}` → CCTV 설정 탭에서:
- go2rtc URL 입력: `http://{rp4_ip}:1984`
- 채널별 RTSP URL 등록 → Backend가 go2rtc API에 자동 반영

---

## Step 5. 센서·릴레이 설정 (관리자가 대시보드에서 진행)

설치 담당자가 RP4에 GPIO 센서/릴레이를 연결한 뒤, 관리자에게 설정 요청:

```
관리자: /stores/{store_id} → 센서·릴레이 탭
    → "HA에서 Entity 가져오기" 클릭
    → MQTT로 RP4에 entity 목록 요청 → 응답 수신 (최대 10초)
    → 각 entity에 대해:
        - 사용자 명칭 지정 (예: "출입구 도어", "냉장고 1")
        - 타입 선택 (출입문 / 냉장고 / 카운터 / 기타)
        - 알림 트리거 여부 설정
    → 저장 → Backend가 monitored_entities MQTT 재발행
    → RP4 Component가 목록 수신 → 해당 entity 상태 모니터링 시작
```

---

## Step 6. 동작 확인 체크리스트

| 항목 | 확인 방법 | 기대 결과 |
|---|---|---|
| Edge 온라인 | 대시보드 매장 상태 | "online" 표시 |
| 센서 상태 수신 | 등록된 센서를 직접 조작 | 대시보드 실시간 상태 변경 |
| 알림 발생 | triggers_alert=1인 센서 조작 | 메인 화면 알림 row 추가 |
| 영상 팝업 | 알림 클릭 | go2rtc 영상 재생 |
| 릴레이 제어 | 대시보드 on/off 버튼 | 실제 릴레이 동작 |
| 오프라인 감지 | RP4 네트워크 연결 해제 90초 대기 | "offline" 표시 |
| 재연결 | RP4 네트워크 재연결 | "online" 복귀 + 상태 재동기화 |

---

## 장애 처리

### Component 등록 실패

| 증상 | 원인 | 조치 |
|---|---|---|
| "store_id not found" 오류 | 관리자가 대시보드에 매장을 등록하지 않음 | 관리자에게 등록 요청 |
| 네트워크 오류 | RP4에서 Backend 서버 미접근 | 네트워크/방화벽 확인 |
| 이미 등록된 store_id | 다른 기기가 동일 store_id 사용 중 | 관리자에게 확인 |

### HA Entity 조회 실패 (503)

| 증상 | 원인 | 조치 |
|---|---|---|
| 10초 timeout | Component가 MQTT에 미연결 | RP4 Component 상태 확인, MQTT 연결 확인 |
| Entity 목록 비어있음 | GPIO 센서가 HA에 미등록 | HA에서 센서 통합 설정 확인 |

### 영상 재생 안됨

| 증상 | 원인 | 조치 |
|---|---|---|
| 영상 팝업에서 로딩 실패 | 관제실 PC에서 RP4 IP 미접근 | 방화벽 규칙 확인 (관제실 IP → RP4 1984/8555 허용) |
| RTSP 스트림 없음 | DVR RTSP URL 오류 | go2rtc 웹 UI에서 스트림 직접 확인 |

---

## 네트워크 포트 개방 요건

| 방향 | 포트 | 프로토콜 | 용도 |
|---|---|---|---|
| RP4 → 중앙 서버 | 8883 | TCP | MQTT TLS |
| RP4 → 중앙 서버 | 443/80 | TCP | Backend API (등록) |
| 관제실 PC → RP4 | 1984 | TCP | go2rtc API / HLS |
| 관제실 PC → RP4 | 8555 | TCP/UDP | go2rtc WebRTC |
