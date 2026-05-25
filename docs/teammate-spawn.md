# tmux × Claude Code teammate spawn 가이드

오케스트레이터(main 워크트리에서 도는 Claude Code 세션)가 다른 팀(backend / frontend / edge)에 작업을 분배할 때 쓰는 표준 방식.

> 검증 환경: tmux 3.4, Claude Code 2.1.150, Ubuntu 24.04.
> 워크트리: `~/production/pcbang-sensor-{backend,frontend,edge}` (각각 `dev/{backend,frontend,edge}`)

---

## 0. 권장 방식 한눈에

**공식 Agent Teams 기능 (v2.1.32+) 사용.** Claude Code가 tmux 자동 분할, 메일박스, 공유 task list, idle 알림, 자연어 종료까지 다 처리. 우리가 손으로 `tmux send-keys` 칠 필요 없음.

```
오케스트레이터(이 세션)
  ├─ TeamCreate("ux-sprint-phase1")            ← 팀 + 공유 task list 생성
  ├─ Agent(team_name=..., name=backend-pcbang) ← 백엔드 teammate, tmux pane 자동 분할
  ├─ Agent(team_name=..., name=frontend-pcbang)← 프론트 teammate, 추가 pane 자동 분할
  ├─ TaskCreate / TaskUpdate(owner=...)         ← 공유 task list로 분배
  ├─ SendMessage(to="backend-pcbang", ...)      ← 직접 메시지
  ├─ (자동) idle_notification 도착              ← teammate가 turn 끝낼 때마다
  ├─ SendMessage({type:"shutdown_request"})    ← 우아한 종료
  └─ TeamDelete()                               ← 디렉토리 + 워크트리 cleanup
```

## 1. 사전 조건

### 1-1. 활성화

`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 필요. 이 프로젝트는 이미 `.claude/settings.json`에 박혀있음:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

`echo $CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 가 `1` 이어야 작동.

### 1-2. 워크트리

PC당 최초 1회:

```bash
cd ~/production/pcbang-sensor-project
git worktree add ~/production/pcbang-sensor-backend  dev/backend
git worktree add ~/production/pcbang-sensor-frontend dev/frontend
git worktree add ~/production/pcbang-sensor-edge     dev/edge

for w in backend frontend edge; do
  ( cd ~/production/pcbang-sensor-$w && git submodule update --init --recursive )
done
```

### 1-3. tmux 세션

오케스트레이터는 반드시 tmux 세션 안에서 실행. tmux 안에 있으면 `teammateMode: "auto"` 가 자동으로 split-pane 모드로 들어감.

---

## 2. 작업 분배 표준 루프

### 2-1. 팀 만들기

```
TeamCreate({
  team_name: "ux-sprint-phase1",
  description: "UX 스프린트 Phase 1 — T1·T2·T3·T4a·T6",
  agent_type: "orchestrator"
})
```

→ `~/.claude/teams/ux-sprint-phase1/config.json` + `~/.claude/tasks/ux-sprint-phase1/` 자동 생성.

### 2-2. teammate spawn (tmux pane 자동 분할)

```
Agent({
  subagent_type: "general-purpose",
  team_name: "ux-sprint-phase1",
  name: "backend-pcbang",
  description: "Backend teammate — Phase 1 백엔드 작업",
  prompt: """
    너는 ux-sprint-phase1 팀의 backend-pcbang teammate.
    너의 작업 디렉토리는 /home/multion/production/pcbang-sensor-backend.
    모든 Bash 호출은 `cd /home/multion/production/pcbang-sensor-backend && ...` 패턴으로 시작.
    절대 다른 워크트리 (frontend/, edge/, main project root) 의 파일을 수정하지 마.

    작업 시작 전:
    1. cd /home/multion/production/pcbang-sensor-backend
    2. cd shared && git pull origin main && cd ..
    3. shared/backend/status.md 와 shared/orchestrator/plan_*.md 의 락인 결정 읽기
    4. 그 후 TaskList로 할당된 task 확인

    완료 후 TaskUpdate로 상태 갱신, SendMessage로 team-lead에게 보고.
  """
})
```

옵션:
- `name` — 메시지 라우팅에 쓰이는 키. 예측 가능한 이름 부여 (`backend-pcbang`, `frontend-pcbang`).
- `subagent_type` — `general-purpose` 면 모든 도구 사용 가능. read-only 작업이면 `Explore`.
- **cwd 옵션은 없음** — prompt에 명시적으로 cd 지시 + 매 Bash 호출 `cd <worktree> && ...` 패턴 강제. 위 확인 결과 매 셸 호출 후 cwd가 리더 cwd(project root)로 리셋되므로 절대경로 또는 매번 cd가 필요함.

spawn 직후 결과:
- `~/.claude/teams/<team>/config.json` `members[]` 에 추가됨 (`tmuxPaneId: "%N"` 자동 기록)
- 같은 tmux window 안에 새 pane이 자동 분할 (별도 split-window 명령 불필요)
- 새 pane 안에서 그 teammate가 spawn prompt 받고 즉시 작업 시작

### 2-3. task 분배 (공유 task list)

```
TaskCreate({
  subject: "T2 backend — alert.new 페이로드에 type_id 추가",
  description: "event_processor.py:170 ... 변경. type_name은 이미 포함됨. ..."
})
TaskUpdate({ taskId: "1", owner: "backend-pcbang", status: "in_progress" })
```

teammate가 idle 상태에서 자동으로 `TaskList` 폴링 + 미할당·미차단 task 자체 claim 도 가능. 명시 할당이 더 deterministic.

### 2-4. 직접 메시지

```
SendMessage({
  to: "backend-pcbang",
  summary: "T2 락인 결정",
  message: "T2 카테고리 필터는 type_id 기준. alert.new에 type_id 추가만 하면 됨. type_name은 이미 있음."
})
```

teammate는 mailbox에서 자동 수신. 응답은 conversation의 다음 turn에 user 메시지처럼 자동 도착 — 폴링 불필요.

### 2-5. idle 알림 자동 수신

teammate가 turn을 끝낼 때마다 `idle_notification` 자동 전송됨. inbox JSON 파일(`~/.claude/teams/<team>/inboxes/team-lead.json`)에도 기록. idle == 멈춤 아니라 "응답 대기" 상태이므로 추가 메시지 보내면 깨어남.

### 2-6. teammate 종료

우아한 종료:

```
SendMessage({
  to: "backend-pcbang",
  message: { "type": "shutdown_request", "reason": "Phase 1 backend 작업 완료" }
})
```

teammate가 승인하면(`shutdown_approved`):
- 해당 tmux pane 자동 닫힘
- config.json `members[]` 에서 제거
- inbox JSON에 `shutdown_approved` 기록 남음

거절 시 (`shutdown_response.approve: false`) — 사유와 함께 거절. 보통 진행 중 작업 마무리 후 다시 시도.

### 2-7. 팀 cleanup

모든 teammate 종료된 뒤:

```
TeamDelete()
```

→ `~/.claude/teams/<team>/` + `~/.claude/tasks/<team>/` 디렉토리 삭제. 활성 멤버 있으면 실패하므로 반드시 shutdown 먼저.

> **주의**: 항상 리더(오케스트레이터)가 TeamDelete 호출. teammate가 호출하면 컨텍스트 꼬임.

---

## 3. 실제 검증 결과 (2026-05-25 testing)

```
TeamCreate("pcbang-test") → ✅ 디렉토리 자동 생성
Agent(team_name=pcbang-test, name=backend-test, prompt=cd+pwd+report)
  → ✅ tmux pane %2 자동 분할 (수동 split-window 불필요)
  → ✅ teammate가 cd 명령 수행 후 "pwd+branch report" 전송
SendMessage({type:"shutdown_request"})
  → ✅ teammate가 shutdown_approved 응답 → pane %2 자동 닫힘
TeamDelete() → ✅ teams/ + tasks/ 디렉토리 모두 정리됨
```

확인된 동작:
- tmux 자동 분할 ✓
- 메일박스 자동 라우팅 ✓
- idle_notification 자동 ✓
- shutdown 우아한 처리 ✓
- TeamDelete cleanup ✓

확인된 제약:
- Agent 도구에 cwd 인자 없음 → prompt에 명시 + 매 Bash 호출 `cd && ...` 패턴 강제
- 매 셸 호출 후 cwd가 리더 cwd로 리셋됨 (검증 출력: `Shell cwd was reset to ...`)

---

## 4. 흔한 함정

| 함정 | 해결 |
|---|---|
| teammate가 다른 워크트리 파일을 건드림 | prompt 첫 줄에 strict 경계 명시 ("절대 X 디렉토리 밖 수정 금지") + 매 Bash 호출 `cd <my-worktree> && ...` |
| 같은 shared submodule에 동시 push race | 모든 shared 변경은 오케스트레이터가 중앙화. teammate는 shared 읽기만, push는 리더에게 위임 |
| teammate가 너무 빨리 idle 처리하고 멈춤 | hooks의 `TeammateIdle` 으로 exit code 2 보내 계속 작동시키거나, 명시적 follow-up 메시지 |
| 토큰 비용 폭증 | 팀원당 컨텍스트 윈도우 별도 → 활성 인원 수만큼 곱셈. 3~5명 권장. 동시 spawn은 작업 분배가 진짜 병렬 가치 있을 때만 |
| `/resume`로 in-process teammate 복원 안 됨 | tmux 분할 모드는 어느 정도 견고하지만, 세션 재개 후 리더가 죽은 teammate에게 메시지 보낼 수 있음 → 새 spawn 명시 |
| 리더가 작업을 자체 구현 시작 | "팀원 완료까지 기다려"라고 명시 |

---

## 5. 부록 — 수동 tmux 패턴 (fallback / 디버깅용)

Agent Teams가 어떤 이유로 작동 안 할 때, 또는 단순 print 모드만 필요할 때.

### 5-1. 수동 split + 인터랙티브 claude

```bash
tmux split-window -h -d \
  -c /home/multion/production/pcbang-sensor-backend \
  -l '50%' \
  "claude --name backend --permission-mode bypassPermissions"

# 새 pane id
BPANE=$(tmux list-panes -F '#{pane_id} #{pane_current_path}' \
        | awk '$2 ~ /-backend$/{print $1; exit}')

# 메시지 + submit (Enter 키워드는 안 먹음, C-m 필요)
tmux send-keys -t "$BPANE" "어떤 task ..."
tmux send-keys -t "$BPANE" C-m

# 응답 capture (claude UI는 placeholder 잔상 있으니 완료 신호로는 부적합)
tmux capture-pane -t "$BPANE" -p | tail -60

# 종료
tmux send-keys -t "$BPANE" "/exit"
tmux send-keys -t "$BPANE" C-m
```

### 5-2. print 모드 (`-p`) — 자동화 친화

```bash
RESULT=$(cd /home/multion/production/pcbang-sensor-backend && \
  claude -p "단일 task" --permission-mode bypassPermissions)
# 연속 작업: --session-id <UUID> 로 고정 → 후속 --resume <UUID>
```

장점: deterministic. 단점: 시각적 분리 없음. 짧은 자동화에 적합.

### 5-3. 부록 방식의 알려진 제약

- `send-keys ... Enter` 가 일부 환경에서 인식 안 됨 → `C-m` 필수
- `capture-pane` 은 PTY의 "지금 그려진 그림" — placeholder 잔상 자주 보임. 응답 완료 시점 감지는 spinner 키워드(`Caramelizing` 등) polling 또는 시간 대기
- 같은 pane을 사람과 오케스트레이터가 동시에 키 입력하면 텍스트 섞임 — 공식 Agent Teams는 메일박스라 이 문제 없음

이런 한계 때문에 메인 방식은 **Agent Teams** 권장. 부록은 디버깅·예외 상황 전용.

---

## 6. 디버깅 명령

```bash
# 활성 팀 + 멤버
ls ~/.claude/teams/
cat ~/.claude/teams/<team>/config.json | jq '.members[] | {name, tmuxPaneId, isActive}'

# 메일박스 — 누가 누구에게 뭘 보냈는지
ls ~/.claude/teams/<team>/inboxes/
cat ~/.claude/teams/<team>/inboxes/<name>.json | jq '.[-5:]'

# task list
ls ~/.claude/tasks/<team>/

# tmux pane 매핑
tmux list-panes -F '#{pane_id} #{pane_current_command} #{pane_title}'

# 강제 정리 (TeamDelete 실패 시)
tmux kill-pane -t %N      # 해당 pane 죽이기
rm -rf ~/.claude/teams/<team>/ ~/.claude/tasks/<team>/
```

---

## 7. 표준 흐름 요약 (오케스트레이터용 체크리스트)

1. [ ] `shared/orchestrator/plan_*.md` 락인 결정 + `shared/{팀}/status.md` 갱신 + shared push
2. [ ] `TeamCreate` — Phase별 팀
3. [ ] `Agent` — 필요한 teammate spawn (prompt에 워크트리 경계·시작 절차 명시)
4. [ ] `TaskCreate` — task 분해 + `TaskUpdate(owner)` 로 할당
5. [ ] 진행 중 추가 결정 — `SendMessage` 로 직접 전달
6. [ ] teammate 완료 + idle 알림 자동 도착
7. [ ] 검증 후 `SendMessage({type:"shutdown_request"})` 로 우아한 종료
8. [ ] `TeamDelete` — 팀 자원 cleanup
9. [ ] shared push로 결과 통합
