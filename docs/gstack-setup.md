# gstack 설치 가이드

`gstack`은 Claude Code에 헤드리스 브라우저 자동화 / QA / 디자인 리뷰 등 53개 스킬을 추가하는 도구다.
이 프로젝트의 `.claude/skills/`는 `.gitignore`에 등록돼 있어 각자 로컬에서 설치한다.

> CLAUDE.md의 `## gstack` 섹션은 그대로 `/agent-browser` 사용 규칙을 따른다.
> 본 문서는 추가 스킬 사용을 원하는 팀원의 로컬 설정 안내일 뿐, 프로젝트 컨벤션을 바꾸지 않는다.

---

## 사전 요구

- Node.js + npm (이미 있을 것)
- `bun` 런타임
  ```bash
  sudo npm install -g bun
  ```
- Playwright Chromium 시스템 라이브러리 (Ubuntu/Debian 기준, setup이 요구하면 자동 안내됨)
  ```bash
  sudo bunx playwright install-deps chromium
  ```

## 설치

```bash
# 1. gstack 저장소 clone
git clone --single-branch --depth 1 \
  https://github.com/garrytan/gstack.git \
  ~/.claude/skills/gstack

# 2. setup 실행 (스킬 등록 + browse 바이너리 빌드 + Chromium 다운로드)
cd ~/.claude/skills/gstack && ./setup

# 3. 프로젝트 로컬 심볼릭 링크 (이미 .gitignore에 등록됨)
cd <pcbang-sensor-project 루트>
mkdir -p .claude/skills
ln -snf ~/.claude/skills/gstack .claude/skills/gstack
```

## 업데이트

```bash
cd ~/.claude/skills/gstack && git pull && ./setup
# 또는 Claude Code 안에서: /gstack-upgrade
```

## 검증

Claude Code 세션에서 `/browse` 또는 `/qa` 등 gstack 스킬이 노출되면 정상.

## 참고

- 공식 저장소: https://github.com/garrytan/gstack
- 53개 스킬 목록은 setup 출력 마지막의 `linked skills:` 줄을 참조
