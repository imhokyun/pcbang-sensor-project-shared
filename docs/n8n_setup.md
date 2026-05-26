# n8n 셋업 — CTRL ↔ IoT webhook 중계

CTRL 시스템이 매장 데이터(특히 `controlTimes`)를 변경할 때마다 n8n이 우리 IoT 서버로 알림을 중계한다. 본 문서는 n8n에 등록할 두 워크플로우의 endpoint·헤더·본문·검증 절차를 정리한다.

> 관련 문서:
> - 명세 원본: [`shared/20260526_ctrl_server_integration.md`](../20260526_ctrl_server_integration.md)
> - 설계: [`shared/orchestrator/design_20260526_ctrl_sync.md`](../orchestrator/design_20260526_ctrl_sync.md)
> - 아키텍처: [`shared/docs/production-architecture.md`](./production-architecture.md)

---

## 0. 전체 흐름

```
[CTRL 시스템] ──POST──▶ [n8n webhook] ──POST──▶ [IoT 서버 endpoint]
                          (1차 수신)            (2차 전달)
                          n8n.multi-on.co.kr    pcbang-iot-api.multion.synology.me

   ┌─────────────────────────┐
   │ 매장 신규 등록           │  ──▶  https://n8n.multi-on.co.kr/webhook/store/created
   │ 매장 정보·시간·삭제 수정  │  ──▶  https://n8n.multi-on.co.kr/webhook/store/updated
   └─────────────────────────┘

n8n은 두 워크플로우 다 같은 우리 endpoint로 push (우리 쪽이 eventType으로 분기).
```

---

## 1. 우리(IoT 서버) endpoint 명세

### 수신 URL

```
POST https://pcbang-iot-api.multion.synology.me/api/v1/webhooks/ctrl/store
```

> 운영(`https://`) 도메인은 Synology Reverse Proxy 가 `:8080` 으로 매핑.

### 필수 헤더

n8n HTTP Request 노드의 **Headers** 항목에 두 줄 등록.

| 헤더 이름 (Name 열) | 헤더 값 (Value 열) | 비고 |
|---|---|---|
| `Content-Type` | `application/json` | 고정 |
| `X-WEBHOOK-SECRET` | `<공유 시크릿 값>` | **헤더 이름은 정확히 `X-WEBHOOK-SECRET`**. `.env` 변수 이름 `CTRL_WEBHOOK_SECRET` 를 헤더 이름으로 적지 말 것. 양쪽(n8n 입력값 + 우리 `.env` 의 `CTRL_WEBHOOK_SECRET` 값)이 동일해야 함. 불일치/누락 시 `401 INVALID_SECRET`. |

> **흔한 실수**: n8n에서 헤더 이름 칸에 `CTRL_WEBHOOK_SECRET` 을 적고 값 칸에 secret을 넣음. 그러면 우리 backend는 `X-WEBHOOK-SECRET` 헤더를 못 찾아서 401. 이름은 **고정 `X-WEBHOOK-SECRET`**, 값만 `.env` 의 시크릿.

### 본문 (CTRL 원본 payload 그대로 passthrough)

CTRL이 n8n에 보내는 본문을 그대로 우리쪽으로 전달하면 된다. **변형 없이 그대로**.

```json
{
  "storeNo": 30356,
  "storeNm": "강남점",
  "storeAddress": "서울시 강남구 ...",
  "storeSttus": "20",
  "storeSttusNm": "관제",

  "chargerNm": "홍길동",
  "chargerPhone": "010-0000-0000",

  "storePhoneNm":  "대표", "storePhone":  "02-1234-5678",
  "storePhoneNm2": "",     "storePhone2": "",
  "storePhoneNm3": "",     "storePhone3": "",

  "intercomIp": "192.168.0.10", "intercomId": "admin", "intercomPwd": "{암호화된 값}",
  "cctvIp":     "192.168.0.20", "cctvId":     "admin", "cctvPwd":     "{암호화된 값}",
  "lnshIp":     "192.168.0.1",  "lnshId":     "admin", "lnshPwd":     "{암호화된 값}",
  "mgmtIp":     "192.168.0.30", "mgmtCvcrPwd": "{암호화된 값}", "mgmtPtjPwd": "{암호화된 값}",

  "controlTimes": [
    { "daySn": 2, "dayNm": "월", "beginTime": "03:00", "endTime": "09:00" },
    { "daySn": 3, "dayNm": "화", "beginTime": "03:00", "endTime": "10:00" },
    { "daySn": 4, "dayNm": "수", "beginTime": "03:00", "endTime": "09:00" },
    { "daySn": 5, "dayNm": "목", "beginTime": "03:00", "endTime": "10:00" },
    { "daySn": 6, "dayNm": "금", "beginTime": "03:00", "endTime": "10:00" },
    { "daySn": 7, "dayNm": "토", "beginTime": "05:00", "endTime": "11:00" },
    { "daySn": 1, "dayNm": "일", "beginTime": "",      "endTime": ""      }
  ],

  "delAt": "N",
  "eventType": "updated",
  "timestamp": "2026-05-26T12:37:17.874"
}
```

**우리는 이 중 다음만 실제로 사용한다**:
- `storeNo` — 매장 식별자 (우리 DB `stores.store_id` 매칭)
- `eventType` — `created` / `updated` / `time_updated` / `deleted` 분기
- `controlTimes[]` — 요일별 관제시간 (우리 `monitoring_schedules` 반영)

나머지 필드(매장명·주소·담당자·연락처·장비 IP/Pwd 5필드)는 **수신 후 무시**. CTRL이 source-of-truth이므로 IoT DB에 미러링하지 않는다 (보안 표면 좁힘).

### 응답

| HTTP | 의미 |
|---|---|
| `200 OK` | 정상 처리 (eventType 분기 + monitoring_schedules 반영 또는 로그만). n8n은 200만 성공으로 인식. |
| `401 Unauthorized` | `X-WEBHOOK-SECRET` 불일치/누락 |
| `422 Unprocessable Entity` | 본문 JSON 파싱 실패 또는 필수 필드(`storeNo`, `eventType`, `controlTimes`) 누락 |
| `5xx` | 우리 서버 내부 오류 (DB 다운 등). n8n 재시도 없음 — 로그 확인 필요 |

응답 본문 형식:
```json
{ "success": true, "data": { "store_id": 30356, "event_type": "updated", "applied": "schedules", "updated_count": 7, "skipped_manual": 0 } }
```

---

## 2. eventType별 우리 처리 동작

| `eventType` | n8n → 우리 호출 | 우리 처리 |
|---|---|---|
| `created` | 매장 신규 등록 | `controlTimes` → `monitoring_schedules` 반영. **우리 DB에 `stores` row 없으면 무시 + 로그** (Edge 등록 우선). |
| `updated` | 매장 정보 수정 (이름·주소·장비 등) | controlTimes만 추출해 반영. 나머지 필드는 무시. |
| `time_updated` | 관제시간만 수정 | 동일하게 controlTimes 반영. |
| `deleted` | 매장 폐점/소프트 삭제 | **로그만 남기고 DB 변경 X**. 운영자가 IoT 쪽 데이터 정리 여부 수동 결정 (실수 보호). |

> **갱신 정책 (2026-05-26 정책 정정)**: CTRL = source-of-truth. `monitoring_schedules.is_manual` 값과 **무관하게 모든 요일이 CTRL controlTimes 로 갱신**. `is_manual` 컬럼은 그대로 유지(누가 손댔는지 흔적). 응답의 `skipped_manual`은 호환성 위해 키만 남기고 항상 0.

> 매장이 우리 DB에 없는데 webhook이 오면 → `200 OK` 반환 + `store_not_found store_no=N` 로그. n8n은 성공으로 인식하고 끝.

---

## 3. n8n 워크플로우 구성

n8n에 두 개의 워크플로우를 만든다. 둘 다 우리 endpoint 1곳으로 push.

### 워크플로우 A — store/created

| 항목 | 값 |
|---|---|
| 1차 수신 URL (CTRL이 호출) | `https://n8n.multi-on.co.kr/webhook/store/created` |
| 트리거 | Webhook 노드, Method `POST`, Path `store/created` |
| 처리 노드 | HTTP Request 노드 (또는 동등) |
| HTTP Request Method | `POST` |
| HTTP Request URL | `https://pcbang-iot-api.multion.synology.me/api/v1/webhooks/ctrl/store` |
| 헤더 (Name → Value 2줄) | `Content-Type` → `application/json`<br/>`X-WEBHOOK-SECRET` → `{{ $env.CTRL_WEBHOOK_SECRET }}` &nbsp; *(헤더 이름 고정, 값만 n8n 환경변수에서 가져옴)* |
| Body | `{{ $json }}` (트리거 본문 그대로 전달) |
| 성공 응답 | HTTP 200만 성공 |
| 재시도 | 없음 (운영 명세상 1회). 실패는 n8n 실행 로그에서 확인. |

### 워크플로우 B — store/updated

| 항목 | 값 |
|---|---|
| 1차 수신 URL (CTRL이 호출) | `https://n8n.multi-on.co.kr/webhook/store/updated` |
| 트리거 | Webhook 노드, Method `POST`, Path `store/updated` |
| 처리 노드 | HTTP Request 노드 |
| HTTP Request Method | `POST` |
| HTTP Request URL | `https://pcbang-iot-api.multion.synology.me/api/v1/webhooks/ctrl/store` |
| 헤더 (Name → Value 2줄) | `Content-Type` → `application/json`<br/>`X-WEBHOOK-SECRET` → `{{ $env.CTRL_WEBHOOK_SECRET }}` &nbsp; *(헤더 이름 고정, 값만 n8n 환경변수에서 가져옴)* |
| Body | `{{ $json }}` |
| 성공 응답 | HTTP 200만 성공 |
| 재시도 | 없음 |

> updated 워크플로우는 본문의 `eventType`이 `updated` / `time_updated` / `deleted` 세 가지로 들어온다. 분기는 우리 쪽에서 함. n8n에서 분기할 필요 없음.

---

## 4. 공유 시크릿 (`CTRL_WEBHOOK_SECRET`)

### 생성

운영 서버에서 1회 실행:

```bash
openssl rand -hex 32
# 예: 3f9a8c5d1e7b4a2f6c8d9e0a1b3c4d5e7f8a9b0c2d3e4f5a6b7c8d9e0f1a2b3
```

### 등록 위치

| 위치 | 키 | 값 |
|---|---|---|
| n8n | 환경 변수 `CTRL_WEBHOOK_SECRET` (Settings → Variables) | 위 생성값 |
| 운영 서버 `/opt/.../backend/.env` | `CTRL_WEBHOOK_SECRET=` | **동일한 값** |

**둘이 정확히 일치해야 한다**. 한 글자 어긋나면 n8n → 우리 호출이 전부 401.

### 회전 절차

키 노출 의심 시:
1. 새 키 생성: `openssl rand -hex 32`
2. 우리 `.env` 갱신 → backend systemd restart
3. n8n 변수 갱신
4. (선택) CTRL 에서 매장 1건 dry-run 으로 검증

순서가 어긋나면 잠시 401 발생. 사용자 영향 최소화 위해 한밤중 또는 정기 점검 시간에 회전.

---

## 5. 검증 (dry-run)

### 5-1. 직접 curl로 우리 endpoint 호출 (n8n 미경유)

backend 구현 완료 + 운영 배포 후:

```bash
# 운영 .env의 CTRL_WEBHOOK_SECRET 값 사용
SECRET="<위에서 등록한 값>"

curl -X POST https://pcbang-iot-api.multion.synology.me/api/v1/webhooks/ctrl/store \
  -H "Content-Type: application/json" \
  -H "X-WEBHOOK-SECRET: $SECRET" \
  -d '{
    "storeNo": 30584,
    "storeNm": "테스트매장",
    "eventType": "time_updated",
    "controlTimes": [
      { "daySn": 2, "dayNm": "월", "beginTime": "03:00", "endTime": "09:00" },
      { "daySn": 3, "dayNm": "화", "beginTime": "03:00", "endTime": "10:00" },
      { "daySn": 4, "dayNm": "수", "beginTime": "03:00", "endTime": "09:00" },
      { "daySn": 5, "dayNm": "목", "beginTime": "03:00", "endTime": "10:00" },
      { "daySn": 6, "dayNm": "금", "beginTime": "03:00", "endTime": "10:00" },
      { "daySn": 7, "dayNm": "토", "beginTime": "05:00", "endTime": "11:00" },
      { "daySn": 1, "dayNm": "일", "beginTime": "",      "endTime": ""      }
    ],
    "delAt": "N",
    "timestamp": "2026-05-26T12:00:00.000"
  }'
```

기대:
- HTTP `200`
- 응답에 `updated_count` 포함 (대상 매장의 `is_manual=0` 행 수만큼)
- DB 확인: `SELECT day_of_week, start_time, end_time, is_manual FROM monitoring_schedules WHERE store_id=30584 ORDER BY day_of_week;`

### 5-2. 잘못된 시크릿으로 호출

```bash
curl -X POST https://pcbang-iot-api.multion.synology.me/api/v1/webhooks/ctrl/store \
  -H "Content-Type: application/json" \
  -H "X-WEBHOOK-SECRET: wrong-value" \
  -d '{"storeNo": 30584, "eventType": "updated", "controlTimes": []}'
```

기대: HTTP `401`.

### 5-3. n8n 워크플로우 통합

n8n 워크플로우 등록 후, n8n UI의 "Execute Workflow" 버튼으로 트리거. CTRL에서 실제 매장 controlTimes를 한 번 바꾸는 게 가장 확실:

1. CTRL UI에서 매장 1개 (예: 30584) 의 월요일 관제시간을 `03:00→09:00` → `04:00→08:00` 같이 변경 + 저장
2. n8n 실행 로그 확인: `store/updated` 트리거 발생 → HTTP Request 노드 200 응답
3. 우리 운영 서버 로그 확인: `sudo journalctl -u pcbang-backend -f | grep webhook` 으로 진입 확인
4. 운영 DB 확인: 해당 매장 `day_of_week=0` (월요일) 행의 `start_time/end_time` 갱신 확인

---

## 6. 트러블슈팅

| 증상 | 원인 | 조치 |
|---|---|---|
| n8n에서 401 응답 | (a) 헤더 이름이 `X-WEBHOOK-SECRET` 가 아닌 `CTRL_WEBHOOK_SECRET` 같은 다른 값 → 우리가 헤더를 못 찾아 401. (b) 이름 맞지만 값 불일치 | n8n 노드 Headers 표 첫 칸(Name)이 정확히 `X-WEBHOOK-SECRET` 인지 확인. 그 다음 값(`{{$env.CTRL_WEBHOOK_SECRET}}`) 이 우리 `.env` 와 일치하는지(공백·줄바꿈 포함) 비교. |
| n8n에서 404 응답 | URL 오타 또는 우리 backend `/webhooks` 라우터 미등록 | `curl -I https://pcbang-iot-api.multion.synology.me/api/v1/webhooks/ctrl/store` 응답 405면 라우터 등록은 OK (GET 미허용). 404면 라우터 등록 안 된 상태 → backend 재시작 또는 main.py include_router 확인. |
| n8n에서 5xx | 우리 backend 내부 오류 | `sudo journalctl -u pcbang-backend -n 100` 로 스택트레이스 확인. 흔한 원인: CTRL_WEBHOOK_SECRET 미설정 (`KeyError`). |
| 200 OK인데 monitoring_schedules 미갱신 | store가 우리 DB에 없거나 모든 row가 is_manual=1 | 응답 본문의 `updated_count`·`skipped_manual` 확인. 매장 미존재면 로그에 `store_not_found store_no=N`. 수동 보존된 매장이면 `skipped_manual=N`. |
| controlTimes는 갱신됐는데 요일이 어긋남 | DoW 매핑 버그 | `_ctrl_daysn_to_our_dow(n) = (n+5)%7` 가 단위테스트로 보호되어 있어 발생 가능성 낮음. 그래도 의심되면 `tests/test_ctrl_sync.py` 7요일 케이스 실행. |
| n8n 실행 로그에는 200인데 로그가 누락된 매장이 있음 | webhook 본문에 `storeNo` 또는 `controlTimes` 누락 | n8n 트리거 본문 검사 (n8n UI → execution → input). CTRL이 빈 값으로 보내는 경우 우리는 422 또는 store_not_found 로그. |

---

## 7. 운영 .env 갱신 체크리스트

backend 구현 머지 후 운영 적용:

```bash
# /opt/pcbang/backend/.env (또는 운영 경로) 에 추가
CTRL_API_BASE=https://<ctrl 운영 도메인>
CTRL_API_KEY=byIL129ZEUb9ySOy6iZPcr0lPCpS1ug3wi72QW4CWCBRfQt6mqRXSrV9HGXtZJHK
CTRL_WEBHOOK_SECRET=<openssl rand -hex 32 결과>
```

- `CTRL_API_BASE` — CTRL 운영팀에 확인 필요 (`/api/store/store_detail` 호출 base URL)
- `CTRL_API_KEY` — 명세 문서에 적힌 값 그대로 (`byIL...QW4...`). 키 회전 시 CTRL 관리자가 발급.
- `CTRL_WEBHOOK_SECRET` — 우리가 생성, n8n에도 동일하게 등록

`.env` 갱신 후:
```bash
sudo systemctl restart pcbang-backend
sudo journalctl -u pcbang-backend -n 50 | grep -E "CTRL|webhook"
```

---

## 8. 보안 주의

- `X-SERVER-API-KEY` (CTRL 인증) + `X-WEBHOOK-SECRET` (n8n→우리 인증) 모두 **로그/스크린샷/git 절대 노출 X**
- 본 문서를 외부 공유 시 위 키 자리는 빈 `<...>` 로 대체
- 키 유출 의심 시 §4 회전 절차 실행
- backend 코드 어디서도 secret 값을 logger로 출력하지 않는다 (config.py 에서 마스킹 권장)
