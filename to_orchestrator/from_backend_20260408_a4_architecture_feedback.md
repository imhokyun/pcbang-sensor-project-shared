# [Backend → Orchestrator] A4 장시간 개방 감지 — 아키텍처 변경 제안

**발신**: Backend  
**수신**: Orchestrator  
**일시**: 2026-04-08

---

## 제안 요약

A4-BE(Backend 백그라운드 폴링)를 **Edge 감지 + Backend 수신** 방식으로 변경을 요청합니다.

---

## 현재 계획의 문제점

현재 계획은 Backend가 1분마다 DB를 폴링하여 장시간 개방 sensor를 감지합니다.

매장이 늘어날수록 Backend에 부하가 집중됩니다:

```
매장  10개 → 1분마다 DB 쿼리 10회
매장 100개 → 1분마다 DB 쿼리 100회
매장 500개 → 1분마다 DB 쿼리 500회
```

또한 최대 1분의 감지 지연이 발생하고, `EntityType.threshold_minutes` 컬럼을 DB에 추가·관리해야 합니다.

---

## 제안 아키텍처

```
[Edge / HA]                         [Backend]              [Frontend]
  binary_sensor on 상태 유지
  HA automation 타이머 트리거
  (X분 경과 시 정확히 발화)
          │
          └──MQTT publish──────────▶ 토픽 구독
                                     AlertEvent 생성
                                     WS broadcast ────────▶ 알림 표시
```

**MQTT 토픽 (제안)**:
```
pcbang/{store_id}/entities/{ha_entity_id}/long_open

payload: {
  "ha_entity_id": "binary_sensor.naengjanggo_1",
  "duration_minutes": 5,
  "triggered_at": "2026-04-08T14:32:00+00:00"
}
```

---

## 방식별 비교

| 항목 | Backend 폴링 | Edge 감지 (제안) |
|------|-------------|----------------|
| 매장 100개 시 부하 | 매분 100회 DB 쿼리 | Backend 변화 없음 |
| 감지 정확도 | 최대 1분 오차 | 초 단위 정밀 |
| 확장성 | Backend 병목 | 수평 확장 자동 |
| 임계값 관리 | DB 컬럼 추가 필요 | Edge 로컬 설정 |
| Backend 구현 | 폴링 서비스 + DB 컬럼 | MQTT 구독 추가만 |

---

## 변경 시 각 팀 작업량

| 팀 | 현재 계획 | 변경 후 |
|----|---------|--------|
| Backend | DB 마이그레이션 + 폴링 서비스 + scheduler 등록 | MQTT 토픽 구독 추가만 |
| Edge | 선택(Optional) | 필수 — HA automation 구현 |
| Frontend | 변화 없음 | 변화 없음 |

Edge 작업량이 늘어나는 단점이 있으나, 전체 시스템 확장성 측면에서 더 나은 구조입니다.

---

## 요청사항

1. 위 아키텍처로 계획 수정 검토 부탁드립니다.
2. Edge 팀 의견 확인 후 결정해 주시면 감사합니다.
   (저희도 별도로 Edge 팀에 문의 문서를 발송합니다.)
3. 확정 시 MQTT 계약 문서(`contracts/mqtt.md`) 업데이트가 필요합니다.
