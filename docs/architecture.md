# 시스템 아키텍처

## 전체 구성도

```
┌─────────────────────────────────────────────────────────┐
│                     중앙 서버                            │
│                                                         │
│   ┌──────────────┐        ┌─────────────────────┐      │
│   │  Mosquitto   │◄──────►│   FastAPI Backend   │      │
│   │ (MQTT Broker)│  sub   │  + WebSocket Server │      │
│   └──────────────┘        └──────────┬──────────┘      │
│          ▲                           │ WS / REST        │
│          │ MQTT (TLS)                ▼                  │
│          │                  ┌────────────────┐          │
│          │                  │  Next.js       │          │
│          │                  │  Dashboard     │          │
│          │                  └────────────────┘          │
└──────────┼──────────────────────────────────────────────┘
           │
    ┌──────┴──────┐      ┌──────────────┐      ┌──────────────┐
    │  매장 A     │      │  매장 B      │      │  매장 N      │
    │  RP4        │      │  RP4         │      │  RP4         │
    │             │      │              │      │              │
    │ HA OS       │      │ HA OS        │      │ HA OS        │
    │ + Custom    │      │ + Custom     │      │ + Custom     │
    │   Component │      │   Component  │      │   Component  │
    │             │      │              │      │              │
    │ go2rtc      │      │ go2rtc       │      │ go2rtc       │
    │ (RTSP→WebRTC│      │              │      │              │
    │             │      │              │      │              │
    │ [DVR RTSP]  │      │ [DVR RTSP]   │      │ [DVR RTSP]   │
    └─────────────┘      └──────────────┘      └──────────────┘
```

## 컴포넌트별 기술 스택

| 컴포넌트 | 기술 | 비고 |
|---|---|---|
| Edge OS | Home Assistant OS | RP4 전용 이미지 |
| Edge Integration | Python (HA Custom Component) | GPIO → MQTT |
| Edge Stream | go2rtc | RTSP → WebRTC/HLS |
| MQTT Broker | Eclipse Mosquitto | 중앙 서버, TLS 권장 |
| Backend API | FastAPI (Python) | MQTT subscriber + REST + WS |
| Database | PostgreSQL (Docker) | 이벤트 이력, MQTT 인증 |
| Frontend | Next.js (React) | App Router |
| 실시간 통신 | WebSocket (FastAPI) | 또는 SSE |

## 네트워크 포트

| 서비스 | 포트 | 프로토콜 |
|---|---|---|
| Mosquitto | 1883 (내부) / 8883 (TLS) | MQTT |
| FastAPI | 8080 | HTTP/WS |
| Next.js | 3000 | HTTP |
| go2rtc API | 1984 | HTTP |
| go2rtc WebRTC | 8555 | UDP/TCP |
| PostgreSQL | 5432 | TCP (내부) |

## 영상 스트림 접근 방식

```
Frontend → go2rtc (Edge) 직접 접근 (iframe)
  URL: http://{edge_ip}:1984/stream.html?src={stream_source}

예: http://192.168.0.19:1984/stream.html?src=ch1_sub
```

- **stream_source**: go2rtc 내부 소스명 (예: `ch1_sub`, `ch2_sub`)
- **stream_url**: Backend가 `go2rtc_url + /stream.html?src= + stream_source` 조합하여 응답에 포함
- Frontend는 `camera.stream_url` 또는 `alert.stream_url`을 iframe src로 직접 사용
- go2rtc가 내부적으로 RTSP → WebRTC/HLS 변환 처리 (Frontend는 iframe만 사용)
- 조건: Frontend가 Edge IP에 직접 접근 가능한 네트워크 구성 (LAN 또는 VPN)
