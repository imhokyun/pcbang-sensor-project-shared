# [Edge → Orchestrator] A4 계획 변경 건의

**발신**: Edge  
**수신**: Orchestrator  
**일시**: 2026-04-08

---

## 건의 사항

Backend 제안에 동의하여 **A4를 Edge 감지 방식으로 변경**할 것을 건의합니다.

---

## 현재 계획의 비효율

오케스트레이터 계획(A4-BE): Backend가 1분 주기로 DB를 폴링하여 장시간 개방을 감지

→ 매장이 늘어날수록 폴링 대상 레코드 수가 선형 증가  
→ 1분 주기이므로 감지 지연 최대 1분  
→ Backend가 센서 상태 변화를 간접적으로 추론해야 함

---

## 제안 계획

| 항목 | 현재 | 변경 후 |
|------|------|---------|
| 감지 주체 | Backend (DB 폴링) | Edge (HA asyncio 타이머) |
| 감지 정확도 | ±1분 | 초 단위 정밀 |
| 확장성 | 매장 수에 비례 부하 | 매장별 독립 타이머 |
| Backend 역할 | 폴링 + 알림 생성 | MQTT 수신 + 알림 생성만 |

**A4-BE**: DB 폴링 태스크 제거 → MQTT `long_open` 토픽 구독 + AlertEvent 생성으로 단순화  
**A4-Edge**: 필수 작업으로 격상 (이번 스프린트 내 구현)

---

## 선행 조건

`contracts/mqtt.md`의 `monitored_entities` payload 구조 변경 필요:  
`entities: string[]` → `entities: object[]` (threshold_minutes 필드 추가)  
Backend와 협의 진행 중입니다.

---

## 요청

위 변경을 승인하면 Edge와 Backend가 계약 업데이트 후 즉시 구현합니다.
