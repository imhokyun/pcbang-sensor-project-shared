# [Backend → Frontend] 카메라 API 구조 변경

**발신**: Backend  
**수신**: Frontend  
**일시**: 2026-04-07

---

## 변경 내용

카메라 API에서 `rtsp_main`, `rtsp_sub` 필드가 제거되고, go2rtc 스트림 구조로 단순화됐습니다.

### 변경 전
```json
{
  "channel": 1,
  "name": "입구",
  "rtsp_main": "rtsp://...",
  "rtsp_sub": "rtsp://...",
  "stream_name": "30584_ch01"
}
```

### 변경 후
```json
{
  "channel": 1,
  "name": "입구",
  "stream_source": "ch1_sub",
  "stream_url": "http://localhost:1984/stream.html?src=ch1_sub"
}
```

---

## stream_url 구조

- `store.go2rtc_url` : go2rtc 서버 주소 (매장별, 수정 가능)
- `camera.stream_source` : go2rtc 소스명 (채널별)
- `stream_url` = `go2rtc_url` + `/stream.html?src=` + `stream_source` ← Backend가 조합하여 응답에 포함

**`stream_url`을 iframe 또는 img src에 바로 사용하면 됩니다.**

---

## 현재 강남점(30584) 카메라 목록

| channel | stream_source | stream_url |
|---------|--------------|-----------|
| 1 | ch1_sub | http://localhost:1984/stream.html?src=ch1_sub |
| 2 | ch2_sub | http://localhost:1984/stream.html?src=ch2_sub |
| 3 | ch3_sub | http://localhost:1984/stream.html?src=ch3_sub |
| 4 | ch4_sub | http://localhost:1984/stream.html?src=ch4_sub |
| 5 | ch5_sub | http://localhost:1984/stream.html?src=ch5_sub |
| 6 | ch6_sub | http://localhost:1984/stream.html?src=ch6_sub |
| 7 | ch7_sub | http://localhost:1984/stream.html?src=ch7_sub |

---

## 카메라 등록/수정 API

**POST /api/v1/stores/{store_id}/cameras**
```json
{ "channel": 1, "name": "입구", "stream_source": "ch1_sub" }
```

**PUT /api/v1/stores/{store_id}/cameras/{channel}**
```json
{ "name": "정문", "stream_source": "ch1_sub" }
```

---

## go2rtc_url 수정

매장의 go2rtc 주소가 바뀌면 (운영서버 이전 등):
```
PUT /api/v1/stores/{store_id}
{ "go2rtc_url": "http://새주소:1984" }
```
→ 모든 채널의 `stream_url`이 자동으로 새 주소를 반영합니다.
