# pcbang-sensor-project-shared

[pcbang-sensor-project](https://github.com/imhokyun/pcbang-sensor-project)의 공유 저장소.

`backend/`, `frontend/`, `edge/` 에 git submodule로 포함되어 에이전트 간 계약 문서와 통신을 담당한다.

---

## 구조

```
shared/
├── contracts/
│   ├── mqtt.md          # MQTT topic/payload 계약, Provisioning 흐름
│   └── api.md           # REST/WebSocket API 계약
├── docs/
│   ├── project-vision.md
│   ├── architecture.md
│   ├── db-schema.md
│   ├── frontend-screens.md
│   └── ...
├── backend/
│   ├── status.md        # ← 에이전트 시작 시 유일한 진입점
│   ├── request/         # 다른 팀이 보낸 요청 (미검토)
│   ├── todo/            # 검토 완료, 작업 진행 중
│   └── done/            # 완료된 태스크
├── frontend/
│   ├── status.md
│   ├── request/ / todo/ / done/
├── edge/
│   ├── status.md
│   ├── request/ / todo/ / done/
├── orchestrator/
│   ├── status.md
│   ├── request/ / todo/ / done/
└── PROJECT_STATUS.md    # 전체 스프린트 현황 (Orchestrator 유지)
```

---

## 에이전트 시작 시 읽기 순서

```
1. shared/{내 팀}/status.md  ← 유일한 진입점
2. shared/{내 팀}/todo/ 파일 ← status.md가 가리키는 것만
3. shared/contracts/api.md, contracts/mqtt.md  ← 항상
4. shared/docs/  ← 필요한 파일만 (매 세션 전체 읽지 말 것)
```

**절대 읽지 말 것**: `done/` 폴더의 파일 (완료된 작업, 구버전 파일)

---

## 태스크 상태 흐름

```
[다른 팀이 생성]    [내 팀 검토·수락]    [작업 완료]
  request/  →  git mv  →  todo/  →  git mv  →  done/
```

| 폴더 | 의미 | 에이전트 행동 |
|------|------|------------|
| `request/` | 수신된 요청, 미검토 | 검토 후 수락 시 todo/로 이동 |
| `todo/` | 현재 작업 대상 | 파일 읽고 작업 진행 |
| `done/` | 완료 아카이브 | **읽지 말 것** |

---

## 파일명 규칙

`from_{발신팀}_{주제}_{날짜}.md`

```
from_orchestrator_a4_be_long_open_20260408.md
from_edge_a4_timer_spec_20260409.md
from_frontend_b1_api_request_20260409.md
```

- 각 태스크 = 파일 1개 (여러 태스크 묶지 않음)
- 발신팀 접두사 유지 → 폴더 위치(수신팀) + 파일명(발신팀)으로 소통 추적

---

## 다른 팀에 요청 보내는 방법

```bash
# 1. 상대 팀 request/ 에 파일 생성
# 예: Orchestrator → Backend
cat > shared/backend/request/from_orchestrator_new_feature_20260410.md << 'EOF'
# ...태스크 내용...
EOF

# 2. 상대 팀 status.md 업데이트 (📥 request 섹션에 추가)

# 3. push
cd shared
git add .
git commit -m "orchestrator→backend: 새 기능 요청"
git push origin main
```

---

## Submodule 사용법

```bash
# 최신 내용 반영
cd shared && git pull origin main && cd ..

# 변경사항 push
cd shared
git add .
git commit -m "설명"
git push origin main
cd ..
git add shared
git commit -m "Update shared ref"
git push
```
