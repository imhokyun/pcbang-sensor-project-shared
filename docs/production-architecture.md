# 운영 환경 아키텍처 (Production)

운영 서버 = `192.168.0.70` (Ubuntu 24.04, 사내망)

이 문서는 **"실제로 어디에 뭐가 떠 있고, 외부에서 어떻게 들어오고, 트래픽이 어떤 순서로 흐르는지"** 를 한 장에 담는 것이 목적이다. 설계 단계의 추상 아키텍처는 [architecture.md](./architecture.md), 서비스 전체 비전은 [project-vision.md](./project-vision.md) 참고.

---

## 1. 한눈에 보기

```
                               ┌─────────────────────────────────────────────┐
 [관제실 브라우저]              │              인터넷                          │
       │                       └──┬───────────────────────────────────┬──────┘
       │ HTTPS                    │                                   │
       ▼                          ▼                                   ▼
 ┌─────────────────────┐   ┌─────────────────────┐         ┌──────────────────┐
 │ Synology NAS        │   │ Synology NAS        │         │  사내 라우터       │
 │ pcbang-iot.         │   │ pcbang-iot-api.     │         │  공인 IP          │
 │ multion.synology.me │   │ multion.synology.me │         │ 112.220.103.108  │
 │ (Reverse Proxy)     │   │ (Reverse Proxy)     │         │ :8883 포트포워딩  │
 └──────────┬──────────┘   └──────────┬──────────┘         └────────┬─────────┘
            │ HTTP                    │ HTTP/WS                     │ MQTT/TLS
            │ :3000                   │ :8080                       │ :8883
            ▼                         ▼                             ▼
 ╔═══════════════════════════════════════════════════════════════════════════╗
 ║                 운영 서버 192.168.0.70 (이 호스트)                          ║
 ║                                                                            ║
 ║   ┌─────────────────────────┐    ┌──────────────────────────────────────┐  ║
 ║   │ pm2: pcbang-frontend    │    │ systemd: pcbang-backend              │  ║
 ║   │ Next.js standalone      │    │ uvicorn (FastAPI)                    │  ║
 ║   │ 0.0.0.0:3000            │◀──▶│ 0.0.0.0:8080                         │  ║
 ║   └─────────────────────────┘    │  ├─ REST  /api/v1/*                  │  ║
 ║                                  │  ├─ WS    /ws                        │  ║
 ║                                  │  ├─ MQTT client → 127.0.0.1:1883     │  ║
 ║                                  │  ├─ DB     → 127.0.0.1:5433          │  ║
 ║                                  │  └─ HTTP/WS proxy → Edge go2rtc      │  ║
 ║                                  └──────────┬───────────────────────────┘  ║
 ║                                             │                              ║
 ║   docker network "pcbang"                   │                              ║
 ║   ┌──────────────────────────┐   ┌──────────┴────────────┐                 ║
 ║   │ pcbang-emqx (EMQX 5.8.9) │   │ pcbang-db (PG 16)     │                 ║
 ║   │  127.0.0.1:1883  ← backend│  │ 127.0.0.1:5433        │                 ║
 ║   │  0.0.0.0  :8883  ← Edge   │  │  └─ Backend writer    │                 ║
 ║   │  127.0.0.1:18083 ← admin  │  │  └─ EMQX auth source  │                 ║
 ║   └──────────┬───────────────┘   └───────────────────────┘                 ║
 ╚══════════════│═════════════════════════════════════════════════════════════╝
                │ MQTT/TLS :8883
                ▼
       ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
       │ 매장 A — RP4     │ │ 매장 B — RP4     │ │ 매장 N — RP4     │
       │ Home Assistant OS│ │ Home Assistant OS│ │ Home Assistant OS│
       │ + pcbang_sensor  │ │ + pcbang_sensor  │ │ + pcbang_sensor  │
       │ + go2rtc :1984   │ │ + go2rtc :1984   │ │ + go2rtc :1984   │
       │   ← DVR RTSP     │ │   ← DVR RTSP     │ │   ← DVR RTSP     │
       └──────────────────┘ └──────────────────┘ └──────────────────┘
```

---

## 2. 외부에서 들어오는 경로

운영 환경은 **두 개의 진입점**이 있다 — 사람(브라우저)과 기계(Edge RP4).

### 2-1. 브라우저 → 대시보드

```
관제실 브라우저
  │
  │ ① https://pcbang-iot.multion.synology.me  (Next.js)
  │ ② https://pcbang-iot-api.multion.synology.me  (FastAPI REST + WS)
  ▼
Synology NAS (Reverse Proxy + Let's Encrypt 인증서)
  │
  │ HTTP plain
  ▼
192.168.0.70:3000  (Next.js)
192.168.0.70:8080  (FastAPI)
```

- **HTTPS 종료는 Synology에서** 일어남. 서버 내부는 평문 HTTP/WS.
- 쿠키는 `Domain=.multion.synology.me` 로 발급 → 두 서브도메인 모두에서 공유 (cross-subdomain 세션).
- WebSocket(`wss://...api.../ws`)도 같은 Synology 리버스 프록시 규칙을 타며, `Upgrade`/`Connection` 헤더가 통과되도록 설정되어야 함.

### 2-2. Edge(매장 RP4) → MQTT

```
매장 RP4의 pcbang_sensor 컴포넌트
  │
  │ MQTT/TLS  :8883
  ▼
사내 라우터 공인 IP 112.220.103.108:8883
  │
  │ 포트 포워딩
  ▼
192.168.0.70:8883  (EMQX)
  │
  │ EMQX → PostgreSQL 직접 조회로 인증
  ▼
pcbang-db: mqtt_users 테이블 (store_id / hashed password)
```

- Edge가 사용하는 호스트/포트는 Backend가 발급한다: `.env`의 `EDGE_MQTT_HOST=112.220.103.108`, `EDGE_MQTT_PORT=8883`.
- 등록 절차: Edge → `POST /api/v1/edge/register` (공유 시크릿 `EDGE_REGISTER_SECRET`) → Backend가 mqtt_users 생성 + 위 host/port 응답.

### 2-3. 영상 (go2rtc) — 백엔드 프록시

```
브라우저
  │  https://pcbang-iot-api.multion.synology.me/api/v1/edge/{store_id}/go2rtc/...
  ▼
FastAPI (httpx / websockets)
  │  내부망 http://{edge_ip}:1984/...
  ▼
매장 RP4 go2rtc
  │  RTSP ← DVR
  ▼
브라우저 (WebRTC/HLS 재생)
```

HTTPS 페이지에서 평문 `http://{edge_ip}:1984`를 직접 임베드하면 Mixed Content로 차단되므로, **Backend가 HTTP·WebSocket 모두 프록시**한다 (`backend/app/routers/go2rtc.py`, `services/go2rtc.py`).

---

## 3. 서버 내부 — 무엇이 어떻게 떠 있나

### 3-1. 프로세스 / 컨테이너 매트릭스

| 이름 | 실행 방식 | 바인딩 | 역할 |
|---|---|---|---|
| `pcbang-frontend` | pm2 (`pm2 ls`) | `0.0.0.0:3000` | Next.js standalone (`.next/standalone/server.js`). middleware로 미인증 → `/login` 리다이렉트. |
| `pcbang-backend` | systemd (`pcbang-backend.service`) | `0.0.0.0:8080` | uvicorn `app.main:app`. DB+EMQX healthy 대기 후 기동. |
| `pcbang-emqx` | docker compose (`backend/docker-compose.yml`) | `127.0.0.1:1883`, `0.0.0.0:8883`, `127.0.0.1:18083` | MQTT 브로커. PostgreSQL 직접 인증. |
| `pcbang-db` | docker compose | `127.0.0.1:5433` (host) → 컨테이너 5432 | Backend writer + EMQX 인증 소스. 볼륨 `pgdata`. |

Docker 네트워크는 `pcbang` (bridge). Backend는 호스트 네트워크에 떠있고 컨테이너에는 `localhost` 포트로 접속한다 — 컨테이너 간 통신은 `pcbang` 네트워크 내에서만.

### 3-2. 호스트 포트 노출 범위

| 포트 | 누가 LISTEN | 외부 접근 | 용도 |
|---|---|---|---|
| 22 | sshd | 가능(사내) | 운영 SSH |
| 3000 | next-server | Synology RP를 통해 외부 | Next.js |
| 8080 | uvicorn | Synology RP를 통해 외부 | FastAPI REST/WS |
| 8883 | docker(emqx) | 라우터 포워딩으로 외부 | Edge MQTT/TLS |
| 1883 | docker(emqx) | localhost only | Backend ↔ EMQX |
| 5433 | docker(db) | localhost only | Backend ↔ PG (호스트에서 매핑) |
| 18083 | docker(emqx) | localhost only | EMQX Dashboard |

> **잠금 원칙**: DB·Backend↔EMQX·Admin Dashboard는 절대 외부로 새지 않게 `127.0.0.1` 바인딩.

### 3-3. 디렉토리 레이아웃 (호스트)

```
/home/multion/production/pcbang-sensor-project/   ← 운영 코드 (이 서버)
├── backend/
│   ├── .env                        # 운영 환경변수 (커밋 금지)
│   ├── .venv/                      # uv sync 결과
│   ├── app/                        # FastAPI
│   ├── alembic/                    # DB 마이그레이션
│   ├── deploy/pcbang-backend.service
│   ├── docker-compose.yml          # emqx + db
│   ├── emqx/emqx.conf
│   ├── logs/app.YYYY-MM-DD.log     # 일별 로테이션
│   └── snapshots/YYYY-MM-DD/...    # 알림 발생 시 캡처된 jpg
├── frontend/
│   ├── .env                        # NEXT_PUBLIC_* (빌드 시 번들 포함)
│   └── .next/standalone/server.js  # pm2가 이걸 실행
└── shared/                         # git submodule (계약·문서)
```

---

## 4. 환경변수 핵심 (운영값 기준)

`backend/.env` 일부 — 시크릿은 실제 값 대신 의미만:

```ini
DATABASE_URL=postgresql+asyncpg://pcbang:***@localhost:5433/pcbang

MQTT_HOST=localhost            # Backend → EMQX (1883)
MQTT_PORT=1883

EDGE_MQTT_HOST=112.220.103.108 # Edge에게 알려주는 값 (공인 IP)
EDGE_MQTT_PORT=8883

CORS_ORIGINS=http://localhost:3000,https://pcbang-iot.multion.synology.me,http://192.168.0.19:3000
COOKIE_DOMAIN=.multion.synology.me

SECRET_KEY=***                 # 세션 토큰 서명
EDGE_REGISTER_SECRET=***       # Edge 등록 공유 시크릿
API_BASE_URL=https://pcbang-iot-api.multion.synology.me/api/v1   # go2rtc HTML 프록시 치환용

ADMIN_PASSWORD=***             # admin1~5 초기 비밀번호
```

`frontend/.env` (build-time):

```ini
NEXT_PUBLIC_API_URL=http://localhost:8080      # 같은 호스트에서 동작하므로 localhost
NEXT_PUBLIC_WS_URL=ws://localhost:8080/ws
```

> 프론트는 빌드 시 번들에 굳어진다 — 값을 바꾸려면 `npm run build` 재실행 + static 복사 + pm2 restart.

---

## 5. 트래픽 흐름 시나리오

### 5-1. 관리자가 대시보드를 연다

```
1. 브라우저 → https://pcbang-iot.multion.synology.me
2. Synology RP → 192.168.0.70:3000
3. Next.js middleware: session_id 쿠키 없음 → /login 리다이렉트
4. 로그인 폼 → POST https://pcbang-iot-api.multion.synology.me/api/v1/auth/login
5. Synology RP → 192.168.0.70:8080 → FastAPI auth
6. Set-Cookie: session_id=...; Domain=.multion.synology.me; HttpOnly; Secure; SameSite=Lax
7. 브라우저: 두 서브도메인 모두에 쿠키 인식 → 대시보드 진입
8. WS 연결: wss://pcbang-iot-api.../ws → init 메시지 수신 (미확인 알림 + 매장 상태)
```

### 5-2. 매장 센서가 알림을 만든다

```
1. RP4 GPIO → HA → pcbang_sensor → MQTT publish
   topic: pcbang/{store_id}/entities/{ha_entity_id}/state
   via:   112.220.103.108:8883 → 라우터 → 192.168.0.70:8883 → EMQX

2. Backend MQTT client (127.0.0.1:1883) 수신

3. services/event_processor 파이프라인 (3단계, DB 커넥션 점유 중 HTTP 호출 방지)
   ① 상태 변화 판정 + sensor_event 기록
   ② alert 트리거 조건 매칭 → alert_event 생성 (triggers_alert AND alert_triggers)
   ③ go2rtc 스냅샷 캡처 → backend/snapshots/YYYY-MM-DD/*.jpg → snapshot_url 갱신

4. is_in_schedule 판정 (force_alert 1/0/NULL + monitoring_schedules)

5. WebSocket broadcast → alert.new (snapshot_url 포함)

6. 브라우저: 알림 리스트 최상단 추가 + (관제시간 & 토글 ON) 알림음
```

### 5-3. 운영자가 영상 팝업을 연다

```
1. 알림 클릭 → VideoPopup
2. iframe src = https://pcbang-iot-api.../api/v1/edge/{store_id}/go2rtc/stream.html?src=ch1_sub
3. Synology RP → FastAPI go2rtc router
4. FastAPI → 매장 RP4 http://{edge_ip}:1984/stream.html?... (사내망/VPN 경로)
5. HTML 응답에 박힌 절대 URL을 API_BASE_URL 기준으로 치환
6. WebRTC signaling은 WebSocket 프록시(`websockets` 라이브러리)로 동일하게 중계
7. 브라우저 ↔ go2rtc WebRTC 협상 완료 → RTSP→WebRTC 영상 재생
```

---

## 6. 살아있나 확인 (운영 체크리스트)

```bash
# 1) 프로세스 / 컨테이너
pm2 ls                                            # pcbang-frontend online
sudo systemctl status pcbang-backend              # active (running)
docker compose -f /home/multion/production/pcbang-sensor-project/backend/docker-compose.yml ps
                                                  # pcbang-emqx healthy, pcbang-db healthy

# 2) 엔드포인트
curl -s http://localhost:8080/api/v1/health       # {"success":true,"data":{"status":"ok"}}
curl -sI http://localhost:3000                    # HTTP/1.1 307 → /login (정상)
curl -sI https://pcbang-iot-api.multion.synology.me/api/v1/health
curl -sI https://pcbang-iot.multion.synology.me

# 3) MQTT (외부에서)
openssl s_client -connect 112.220.103.108:8883 -servername 112.220.103.108 </dev/null 2>/dev/null | head -3

# 4) 로그
tail -f /home/multion/production/pcbang-sensor-project/backend/logs/app.$(date +%Y-%m-%d).log
sudo journalctl -u pcbang-backend -f
pm2 logs pcbang-frontend --lines 100

# 5) DB
docker exec -it pcbang-db psql -U pcbang -d pcbang -c "SELECT count(*) FROM alert_events WHERE created_at > now() - interval '1 hour';"

# 6) EMQX 대시보드 (SSH 터널)
# 로컬에서: ssh -L 18083:127.0.0.1:18083 multion@192.168.0.70
# 브라우저: http://localhost:18083
```

---

## 7. 자주 쓰는 운영 명령

```bash
# Backend 재시작 (코드 수정 후)
sudo systemctl restart pcbang-backend

# Backend 의존성 / 마이그레이션
cd /home/multion/production/pcbang-sensor-project/backend
uv sync --no-dev
uv run alembic upgrade head

# Frontend 재빌드 + 재시작 (.env 변경, 코드 변경 시)
cd /home/multion/production/pcbang-sensor-project/frontend
npm ci && npm run build
cp -r .next/static .next/standalone/.next/static
cp -r public .next/standalone/public
pm2 restart pcbang-frontend

# EMQX / DB 재시작
cd /home/multion/production/pcbang-sensor-project/backend
docker compose restart emqx
docker compose restart db

# 전체 코드 업데이트
cd /home/multion/production/pcbang-sensor-project
git pull origin main
git submodule update --recursive
```

---

## 8. 자주 헷갈리는 포인트

| 헷갈림 | 진실 |
|---|---|
| "Backend가 어디 떠 있지?" | 컨테이너가 아니라 **호스트의 systemd** (`pcbang-backend.service`). Docker에 있는 건 EMQX/DB뿐. |
| "Frontend도 컨테이너인가?" | 아니. **pm2**로 Next.js standalone을 직접 실행. |
| "Edge가 어떤 호스트로 MQTT 접속하지?" | `112.220.103.108:8883` (라우터 공인 IP). 사내망 IP 아님. |
| "HTTPS는 어디서 끊기지?" | **Synology NAS의 Reverse Proxy**. 서버 안쪽은 평문 HTTP/WS. |
| "쿠키가 왜 두 서브도메인에서 다 먹지?" | `Domain=.multion.synology.me` 로 발급되기 때문. `SECRET_KEY` 운영값으로 반드시 교체 필요. |
| "1883은 외부에서 못 보는데 왜 떠 있지?" | EMQX가 호스트 1883에 바인딩되지만 `127.0.0.1`만이라 외부 차단. Backend 내부 통신 전용. |
| "MQTT 인증은 어디서?" | **EMQX → PostgreSQL `mqtt_users` 테이블 직접 조회**. 별도 인증 플러그인 데몬 없음. |
| "Frontend의 NEXT_PUBLIC_API_URL이 localhost인 이유?" | Frontend와 Backend가 **같은 서버에 올라가 있어서**. 외부 도메인은 Synology RP에서 알아서 매핑. |
| "snapshot은 어디 저장돼?" | `backend/snapshots/YYYY-MM-DD/*.jpg`. FastAPI의 `StaticFiles`로 서빙. |

---

## 9. 변경 시 유의사항

- **`backend/.env` 변경** → `sudo systemctl restart pcbang-backend`
- **`frontend/.env` 변경** → 반드시 **재빌드** (NEXT_PUBLIC_*는 번들에 굳음)
- **DB 스키마 변경** → Alembic 마이그레이션 추가 → `uv run alembic upgrade head` → 백엔드 재시작
- **CORS 도메인 추가** → `.env` `CORS_ORIGINS` 콤마로 추가 → 백엔드 재시작
- **Synology RP 대상 변경** → 제어판 → 로그인 포털 → 고급 → 리버스 프록시 (WebSocket 헤더 체크박스 확인)
- **EMQX 인증 설정 변경** → `backend/emqx/emqx.conf` 수정 → `docker compose restart emqx`

---

## 참고 문서

| 문서 | 내용 |
|---|---|
| [architecture.md](./architecture.md) | 설계 단계 시스템 구성도, 기술 스택 |
| [project-vision.md](./project-vision.md) | 서비스 목표, 데이터 흐름 |
| [db-schema.md](./db-schema.md) | PostgreSQL DDL |
| [installer_workflow.md](./installer_workflow.md) | Edge(RP4) 등록 절차 |
| `../contracts/mqtt.md` | MQTT topic/payload 계약 |
| `../contracts/api.md` | REST/WebSocket API 계약 |
| `../../backend/DEPLOY.md` | Backend 배포 절차 |
| `../../frontend/DEPLOY.md` | Frontend 배포 절차 |
