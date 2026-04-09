# 프로젝트 현황
_업데이트: 2026-04-09 | 담당: Orchestrator_

---

## 현재 스프린트 (2026-04-09~)

**목표: 운영 환경 동일 셋업에서 전체 통합 테스트 통과 → 실서버(192.168.0.70) 이전**

| ID | 팀 | 상태 | 요약 |
|----|----|------|------|
| QA-BE | Backend | 📥 대기 | CORS·쿠키·EMQX 설정 검증 및 버그 수정 |
| QA-FE | Frontend | 📥 대기 | HTTPS 환경 전체 동작 검증 및 버그 수정 |
| QA-Edge | Edge | 📥 대기 | 외부망 연결·등록·MQTT 전체 흐름 검증 |

### 네트워크 셋업 (개발서버 = 운영 동일 조건)
| 서비스 | 외부 주소 | 내부 (현재 개발서버) |
|--------|-----------|----------------------|
| Frontend | `https://pcbang-iot.multion.synology.me` | `192.168.0.19:3000` |
| Backend API | `https://pcbang-iot-api.multion.synology.me` | `192.168.0.19:8080` |
| MQTT | `112.220.103.108:8883` | `192.168.0.19:8883` |

---

## 완료된 스프린트 (2026-04-08)

| ID | 팀 | 결과 | 요약 |
|----|----|------|------|
| A3 | Frontend | ✅ | 알림음 importance 4·5 강조 소리 |
| A1 | Frontend | ✅ | 팝업 UI — store_name/custom_name/시각 강조 |
| A2 | Frontend | ✅ | 알림 목록 type_name 섹션 분리 |
| A4-BE | Backend | ✅ | MQTT long_open 구독 + AlertEvent 생성 |
| A4-Edge | Edge | ✅ | coordinator.py asyncio 타이머 |
| A4-FE | Frontend | ✅ | 장시간개방 알림 수신 확인 |
| B1-BE | Backend | ✅ | GET /stores 검색·페이지네이션 |
| B1-FE | Frontend | ✅ | 매장 검색 UI + 페이지네이션 |

---

## 다음 단계
- 개발서버 통합 테스트 전 시나리오 통과 후 → 실서버(192.168.0.70) 동일 셋업 후 이전
