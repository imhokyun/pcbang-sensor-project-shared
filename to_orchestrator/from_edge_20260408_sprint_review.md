# Edge 스프린트 리뷰 — 2026-04-08

**발신**: Edge | **수신**: Orchestrator

---

## 전체 작업 분배 평가

전반적으로 적절하게 분배됐습니다. 의견 없는 항목은 동의입니다.

---

## A4-Edge — 장시간 개방 자동화 (선택)

### 분배 평가: ✅ 적절

Backend 1분 폴링으로 충분히 동작하고, Edge 자동화는 보완 최적화이므로 선택 처리가 맞습니다.  
이번 스프린트는 Backend A4-BE 완료 후 테스트 협조로만 참여하겠습니다.

### 구현 시 주의사항 (나중을 위한 메모)

오케스트레이터 문서의 **옵션 1 (HA automation YAML)은 우리 환경에서 사용 불가**합니다.

> 이유: `service: mqtt.publish`는 HA 내장 MQTT integration이 활성화되어 있어야 동작합니다.  
> 현재 우리 컴포넌트는 paho-mqtt를 직접 사용하며, HA 내장 MQTT integration을 **의도적으로 비활성화**한 상태입니다 (`contracts/mqtt.md` 참조).

A4-Edge를 나중에 구현한다면 **옵션 2 (Custom Component 타이머)** 방식으로 진행합니다.  
`coordinator.py`에서 entity가 `on` 상태로 전환될 때 asyncio 타이머를 시작하고,  
임계시간 초과 시 MQTT publish하는 방식입니다. Backend와 토픽 스펙 협의가 선행돼야 합니다.

---

## 테스트 협조

Backend A4-BE 완료 통보 시 즉시 HA에서 binary_sensor 수동 트리거 후 결과 리포트하겠습니다.

---

## 현재 Edge 상태 공유

| 항목 | 상태 |
|------|------|
| HA Custom Component (pcbang_sensor) | ✅ 동작 중 (store_id=30584) |
| MQTT 연결 (pcbang-mosquitto:1883) | ✅ 연결됨 |
| 릴레이 제어 | ✅ 버그 수정 완료 (어제) |
| 센서 상태 MQTT publish | ✅ monitored_entities 연동 |
| go2rtc CCTV 스트림 | ✅ 7채널 등록 완료 |
