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
│   ├── project-vision.md   # 서비스 목표, 전체 흐름
│   ├── architecture.md     # 시스템 구성도, 기술 스택
│   ├── db-schema.md        # 전체 DB 테이블 DDL (PostgreSQL)
│   ├── frontend-screens.md # 화면별 UI 설계
│   └── tbd.md              # 미결 사항
├── to_backend/          # frontend/edge → backend 요청함
├── to_frontend/         # backend/edge → frontend 요청함
└── to_edge/             # backend/frontend → edge 요청함
```

---

## 에이전트 간 통신 규칙

각 에이전트는 자신의 inbox 폴더(`to_{self}/`)만 읽고, 다른 에이전트에게 보낼 파일은 상대 폴더에 생성한다.

**파일명 규칙**: `from_{발신}_{YYYYMMDD}_{주제}.md`

```bash
# 예: backend agent가 frontend에게 API 변경사항 전달
shared/to_frontend/from_backend_20260406_api_change.md

# 작성 후 push
cd shared
git add to_frontend/from_backend_20260406_api_change.md
git commit -m "backend→frontend: API 변경사항 전달"
git push origin main
```

각 에이전트는 작업 시작 시 및 수시로 `git pull origin main`으로 inbox를 확인한다.

---

## Submodule 사용법

```bash
# 최초 초기화
git submodule update --init --recursive

# 최신 내용 반영
cd shared && git pull origin main && cd ..
git add shared && git commit -m "Update shared ref"

# 변경사항 push (shared 내부에서)
cd shared
git add .
git commit -m "Update contracts/docs"
git push origin main
```
