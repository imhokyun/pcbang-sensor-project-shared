# [Frontend → Backend] Alert 생성 미동작 — 재확인 요청

**일자**: 2026-04-07  
**발신**: Frontend 팀  
**수신**: Backend 팀

---

## 현상

`entity.update` WS는 정상 수신되고 있으나, `alert.new`는 한 건도 수신되지 않음.

DB 직접 확인 결과 `alerts` 테이블에 레코드가 하나도 없음:

```bash
GET /api/v1/alerts → { "success": true, "data": [] }
```

---

## 확인된 조건 (alert 생성 조건 모두 충족)

| 항목 | 값 | 상태 |
|------|-----|------|
| store force_alert | 1 (강제 ON) | ✅ |
| entity triggers_alert | 1 | ✅ (jeongmun_senseo, id=21) |
| entity is_active | 1 | ✅ |
| WS entity.update 수신 | 정상 | ✅ |
| MQTT 상태변화 발행 | 정상 (off→on 확인) | ✅ |

---

## 재현 흐름

1. Edge에서 `binary_sensor.jeongmun_senseo` off→on 상태 변화 발생
2. MQTT `pcbang/30584/entities/binary_sensor.jeongmun_senseo/state` 발행됨
3. 백엔드가 `entity.update` WS 브로드캐스트 → 프론트 정상 수신
4. **`alert_events` INSERT 및 `alert.new` WS 브로드캐스트 없음**

---

## 추가 확인 사항

응답 문서에서 언급된 `alert_triggers 매칭` 조건의 의미를 확인해 주세요.

```
GET /api/v1/alert-triggers        → 404 Not Found
GET /api/v1/stores/30584/alert-triggers  → 404 Not Found
```

이 엔드포인트가 존재하지 않고, 현재 entity_types에도 trigger 조건 필드가 없습니다.
alert 생성의 추가 조건이 있다면 어디서 설정하는지 알려주시거나, 해당 조건 없이도 `triggers_alert=1`이면 alert가 생성되도록 수정 요청 드립니다.

---

## 요청

1. 백엔드 MQTT 수신 후 alert 생성 코드의 실행 여부 로그 확인
2. `triggers_alert=1` + `force_alert=1` 조건에서 alert가 생성되지 않는 원인 파악 및 수정
3. 수정 후 `shared/to_frontend/` inbox로 완료 통보
