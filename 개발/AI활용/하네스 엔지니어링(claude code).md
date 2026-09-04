> 하네스 엔지니어링 (Harness Engineering) · Claude Code 기준
> 조사일: 2026-09-04
> 요약: **에이전트 성능은 모델이 아니라 모델을 둘러싼 구조가 결정한다.** 그 구조를 설계하는 일이 하네스 엔지니어링이다.

## 한 줄 정의

```
Agent = Model + Harness
```

모델이 두뇌라면 **하네스는 그 두뇌가 실제로 파일을 읽고 명령을 실행하게 만드는 모든 것**입니다. 도구, 컨텍스트, 메모리, 권한, 검증 루프가 여기 들어갑니다.

핵심 명제는 하나입니다. **"괜찮은 모델 + 좋은 하네스"가 "좋은 모델 + 나쁜 하네스"를 이깁니다.** 같은 모델로 Terminal Bench 2.0 순위를 30위에서 5위로 올린 사례가 근거로 자주 인용됩니다. 모델을 기다리는 대신 실패 모드를 하나씩 설정으로 막는 접근입니다.

---

## 이론 — 다섯 덩어리만

| 층 | 하는 일 | Claude Code에서 |
| --- | --- | --- |
| **에이전트 루프 · 도구** | 사고 → 행동 → 관찰 반복 | 내장 도구 + MCP |
| **컨텍스트** | 무엇을 얼마나 넣을지 | `CLAUDE.md`, 압축, Skills |
| **메모리 · 상태** | 세션을 넘어 남기기 | 파일, Git, Skills |
| **권한 · 샌드박스** | 못 하게 막기 | `settings.json` permissions |
| **검증 · 피드백** | 결과를 다음 입력으로 | Hooks (린트·테스트·타입체크) |

### 신호는 두 방향

| | 언제 | 무엇 |
| --- | --- | --- |
| **피드포워드** | 행동 **전** | `CLAUDE.md`, 도구 설명, 시스템 프롬프트 |
| **피드백** | 행동 **후** | 린트 실패, 테스트 실패, hook 차단 |

둘 중 하나만 있으면 잘 안 됩니다. 규칙만 적어두면 안 지키고, 막기만 하면 왜 막혔는지 모릅니다.

### 래칫 원칙 (Ratchet)

**에이전트가 저지른 실수는 다시 못 하게 설정으로 굳힙니다.** 되돌아가지 않는 톱니처럼 규칙이 한 방향으로만 쌓입니다.

```
같은 실수 1회  →  CLAUDE.md에 규칙 한 줄
같은 실수 반복  →  hook으로 자동 차단
그래도 반복    →  권한에서 아예 제거
```

이게 하네스 엔지니어링의 실제 작업 대부분입니다. 처음부터 완벽한 설정을 짜는 게 아니라 **부딪힌 것만 굳혀 나갑니다.**

### 도구 설계 원칙

- **적고 좋은 도구가 많고 어중간한 도구를 이깁니다.** 메뉴가 길어지면 모델이 선택을 못 합니다
- 도구 설명이 곧 UX입니다. 사람에게 설명하듯 씁니다
- 성공은 침묵, 실패만 보고 — 검증 hook은 통과할 때 아무것도 출력하지 않는 게 좋습니다

---

## 구축 — 일단 돌아가는 최소 구성

순서대로 하면 됩니다. **1번만 해도 절반은 먹고 들어갑니다.**

### 1. `CLAUDE.md` — 프로젝트 루트

가장 효과가 큽니다. **60줄 넘기지 마십시오.** 길어지면 모델이 흘려 읽습니다.

```markdown
# 프로젝트

FastAPI + SQLAlchemy 2.0 async. Python 3.13.

## 명령어
- 테스트: `pytest -q`
- 린트: `ruff check .`

## 규칙
- 커밋 전 반드시 테스트 통과
- 테스트를 주석 처리해서 통과시키지 말 것
- DB 스키마 변경은 마이그레이션 파일로만
```

담을 것은 **명령어 · 금지사항 · 프로젝트 고유 사실** 셋뿐입니다. 일반적인 코딩 상식은 넣지 마십시오.

- 개인 전역 규칙은 `~/.claude/CLAUDE.md`
- 팀 공유는 `./CLAUDE.md` (커밋)

### 2. 권한 — `.claude/settings.json`

매번 승인 누르는 걸 줄이고, 위험한 건 아예 막습니다.

```json
{
  "permissions": {
    "allow": [
      "Bash(pytest *)",
      "Bash(ruff check *)",
      "Bash(git status)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Bash(rm -rf *)"
    ]
  }
}
```

**설정 우선순위** (위가 이김)

```
관리자 설정 → CLI(--settings) → .claude/settings.local.json
           → .claude/settings.json → ~/.claude/settings.json
```

`settings.local.json`은 커밋하지 않는 개인용입니다.

### 3. Hook 한 개 — 편집 후 자동 검사

같은 `.claude/settings.json`에 붙입니다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "ruff check . || true" }
        ]
      }
    ]
  }
}
```

주요 이벤트만 알면 됩니다.

| 이벤트 | 시점 |
| --- | --- |
| `SessionStart` | 세션 시작 |
| `UserPromptSubmit` | 프롬프트 처리 전 |
| `PreToolUse` | 도구 실행 **전** — 차단 가능 |
| `PostToolUse` | 도구 실행 성공 후 |
| `Stop` | 응답 종료 |

**차단은 종료 코드 2입니다.** 스크립트가 `exit 2`를 내면 그 도구 호출이 막히고, `stderr`에 쓴 메시지가 모델에게 전달됩니다.

```bash
#!/bin/bash
input=$(cat)
cmd=$(jq -r '.tool_input.command' <<<"$input")
if [[ "$cmd" == *"--force"* ]]; then
  echo "force push 금지" >&2
  exit 2
fi
exit 0
```

matcher는 `Bash`, `Edit|Write`(파이프), `*`(전체), 정규식을 씁니다.

### 4. Skill 한 개 — 반복 절차 빼내기

**`CLAUDE.md`에 넣기엔 긴 절차**를 여기로 옮깁니다. 본문은 **쓸 때만 로드**되므로 길어도 평소 비용이 0에 가깝습니다.

```
.claude/skills/release/SKILL.md
```

```markdown
---
name: release
description: 릴리스 절차. 버전 태깅부터 배포까지. 릴리스·배포 요청 시 사용
---

1. `pytest -q` 전체 통과 확인
2. CHANGELOG.md 갱신
3. `git tag v{버전}` 후 push
4. CI 통과 확인
```

`/release`로 직접 부르거나, description을 보고 Claude가 알아서 씁니다. `.claude/commands/release.md`도 같은 `/release`를 만듭니다(구형 방식, 계속 동작).

### 5. Subagent 한 개 — 탐색 분리

**컨텍스트를 더럽히는 작업**(대규모 검색, 로그 뒤지기)을 별도 창으로 보냅니다. 결과만 돌아옵니다.

```
.claude/agents/explorer.md
```

```markdown
---
name: explorer
description: 코드베이스 광범위 검색. 어디에 무엇이 있는지 찾을 때
tools: Read, Grep, Glob
model: sonnet
---

파일 위치와 관련 심볼만 보고하십시오. 코드 전문을 붙여넣지 마십시오.
```

`tools`로 권한을 좁히는 게 핵심입니다. 읽기 전용 에이전트는 쓰기 도구를 안 줍니다.

### 6. MCP — 외부 시스템 연결

필요할 때만 합니다. 프로젝트 루트 `.mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://example.com/mcp"
    }
  }
}
```

CLI로도 됩니다.

```bash
claude mcp add --transport http github --scope project https://example.com/mcp
```

**MCP는 마지막에 붙이십시오.** 도구가 늘면 모델의 선택 정확도가 떨어집니다. 없어도 되는 건 안 붙이는 게 낫습니다.

---

## 착수 순서

| 시점 | 할 일 |
| --- | --- |
| **Day 0** | `CLAUDE.md` 작성. 명령어 + 금지사항만 |
| **Week 1** | 반복된 실수를 규칙으로. 권한 allow/deny 정리 |
| **Week 2~4** | 검증 hook, Skill, Subagent 추가 |
| **필요할 때** | MCP |

**거꾸로 하지 마십시오.** MCP부터 붙이고 `CLAUDE.md`가 비어 있는 구성이 제일 흔한 실패입니다.

## 점검 목록

- [ ] `CLAUDE.md`가 60줄 이하인가
- [ ] 테스트·린트 명령이 거기 적혀 있는가
- [ ] 자주 승인 누르는 명령이 `allow`에 있는가
- [ ] `.env` 류가 `deny`에 있는가
- [ ] 편집 후 자동으로 도는 검사가 하나라도 있는가
- [ ] 같은 실수를 두 번 봤는데 아직 규칙이 없는 게 있는가

---

## 용어

| 용어 | 뜻 |
| --- | --- |
| **하네스(Harness)** | 모델을 감싼 실행 환경 전체. 마구(馬具)에서 온 말 |
| **래칫(Ratchet)** | 한 방향으로만 조여지는 톱니. 규칙이 되돌아가지 않게 쌓는 방식 |
| **피드포워드** | 행동 전에 주는 지침 |
| **랄프 루프(Ralph Loop)** | 컨텍스트 창을 넘겨가며 작업을 강제로 이어붙이는 장기 실행 패턴 |
| **스프린트 계약** | 착수 전에 완료 기준을 합의해 두는 것 |

## 더 볼 것

- [Agent Harness Engineering — Addy Osmani](https://addyosmani.com/blog/agent-harness-engineering/) — 개념 정리의 기준점
- [awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering) — 도구·패턴 모음
- [Claude Code 공식 문서](https://code.claude.com/docs/en/hooks) — hooks · subagents · skills · settings
- [하네스 엔지니어링 with 클로드 코드](https://www.hanbit.co.kr/books/%ED%95%98%EB%84%A4%EC%8A%A4-%EC%97%94%EC%A7%80%EB%8B%88%EC%96%B4%EB%A7%81-with-%ED%81%B4%EB%A1%9C%EB%93%9C-%EC%BD%94%EB%93%9C?code=B2817272480) (한빛미디어) — 국내서
