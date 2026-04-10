# [Frontend 검토 요청] go2rtc 스냅샷 저장 및 전달 방식

_from: Backend | 2026-04-10_

---

## 개요

센서 상태 변화(도어 열림 등) 시점에 연결된 카메라의 현재 프레임을 캡처하여  
Backend에 JPEG로 저장하고, 알림 및 이벤트 로그에서 사진을 볼 수 있도록 구현하려 한다.

**프론트 쪽에서 구현에 문제가 없는지, 이 방식이 맞는지 검토 요청.**

---

## 스냅샷 전달 방식

### 이미지 서빙 URL

```
GET /api/v1/snapshots/{YYYY-MM-DD}/{filename}.jpg
```

- 인증 불필요 (StaticFiles 직접 서빙)
- 예: `https://pcbang-iot-api.multion.synology.me/api/v1/snapshots/2026-04-10/30_door1_143022.jpg`

### snapshot_url 포맷

응답에 포함되는 `snapshot_url`은 **상대 경로**:

```
/api/v1/snapshots/2026-04-10/30_door1_143022.jpg
```

프론트에서는 `NEXT_PUBLIC_API_URL`의 origin 부분을 앞에 붙여 `<img src>` 사용:

```ts
const imageUrl = `${process.env.NEXT_PUBLIC_API_URL.replace('/api/v1', '')}${snapshot_url}`
// 또는 snapshot_url이 이미 /api/v1/... 로 시작하므로:
const imageUrl = `https://pcbang-iot-api.multion.synology.me${snapshot_url}`
```

---

## WS alert.new 변경사항

`alert.new` 페이로드에 `snapshot_url` 필드 추가:

```json
{
  "type": "alert.new",
  "data": {
    "id": 123,
    "store_id": 30,
    "store_name": "강남점",
    "entity_name": "출입문 1",
    "message": "출입문 1 열림",
    "is_in_schedule": true,
    "stream_url": "http://192.168.20.30:1984/stream.html?src=ch1_sub",
    "snapshot_url": "/api/v1/snapshots/2026-04-10/30_door1_143022.jpg"
  }
}
```

- `snapshot_url`이 `null`일 수 있음 (카메라 없는 엔티티, go2rtc offline 시)
- 기존 `stream_url`은 그대로 유지

---

## GET /logs 변경사항

로그 응답 items에 `snapshot_url` 필드 추가:

```json
{
  "items": [
    {
      "id": 456,
      "store_name": "강남점",
      "entity_name": "출입문 1",
      "state": "on",
      "triggered_at": "2026-04-10T14:30:22",
      "snapshot_url": "/api/v1/snapshots/2026-04-10/30_door1_143022.jpg"
    }
  ]
}
```

- `snapshot_url` null 가능 → 프론트에서 null 체크 후 이미지 렌더링

---

## 적용 화면

1. **알림 모달** — `alert.new` 수신 시 `snapshot_url` 있으면 이미지 표시
2. **이벤트 로그 테이블** — `GET /logs` 응답의 `snapshot_url` 있으면 썸네일 표시

---

## 보관 정책

- 스냅샷은 Backend에서 **2주간 보관** 후 자동 삭제
- 프론트는 별도 관리 불필요

---

## 검토 요청 사항

1. 위 URL 방식(`NEXT_PUBLIC_API_URL` origin + `snapshot_url`)으로 이미지 렌더링 가능한지
2. `null` 처리 방식 — null이면 이미지 영역 숨김 처리 예정인지
3. 알림 모달 / 로그 테이블 UI에서 사진 표시 구현에 문제될 부분 있는지

피드백 없으면 **2026-04-11부터 구현 진행** 예정.
