# 멀티에이전트 오케스트레이션 구성 v1 (전체 Fable 버전)

작성일: 2026-09-11 · 대상: `C:\work\seohwa-agents` (WSL 경로 `/mnt/c/work/seohwa-agents`) · 상태: **현재 운영 중**

## 결론

현재 구성은 역할 9종(판단 1 · 프론트 개발 3 · 백엔드 개발 3 · 문서작업 1 · 디자인 1)에 리뷰어·QA·인테그레이터가 붙은 worktree 기반 관제 구조이며, **모든 세션이 Fable 한 모델로 실행된다.** 원인은 `agent-run.sh` · `fleet-orch.sh` · `wt-new.sh` 어디에도 `--model` 지정이 없어 Claude Code CLI 기본 모델(Fable)이 그대로 쓰이기 때문이다. 판단·구현·grep·diff·검증 로그 읽기까지 전부 최상위 모델 단가로 처리되므로 토큰 비용이 과다하다. 이 문제를 해결한 것이 v2다.

## 1. 구성 요약

| 구분 | 역할 파일 | 개수 | 실행 위치 | 모델 |
| --- | --- | --- | --- | --- |
| 판단(ORCHESTRATOR) | `roles/orchestrator.md` | 1 | `seohwa-agents` 루트 세션, 또는 `fleet-orch.sh`가 헤드리스로 기동 | Fable |
| 프론트 개발(DEV-FE) | `roles/dev-fe.md` | 3 (동시 상한 `MAX_WT_FE=3`) | `wt/TASK-*/` worktree 세션 | Fable |
| 백엔드 개발(DEV-BE) | `roles/dev-be.md` | 3 (설정값 `MAX_WT_BE=2`, 운영은 3) | `wt/BE-*/` worktree 세션 | Fable |
| 문서작업(OFFICE) | `roles/office.md` | 1 (상한 3) | `wt/OF-*` `wt/GO-*` worktree 세션 | Fable |
| 디자인(DESIGNER) | `roles/designer.md` | 1 | `.fleet/design/<ID>/` 산출 | Fable |
| 리뷰어(REVIEWER) | `roles/reviewer.md` | 카드당 1회 | `agent-run.sh <ID> reviewer` | Fable |
| QA | `roles/qa.md` | 필요 시 | worktree 세션 | Fable |
| 인테그레이터(INTEGRATOR) | `roles/integrator.md` | 1 | 루트 세션 | Fable |

- 역할 파일은 `wt-new.sh`가 worktree 생성 시 `CLAUDE.md`(코드) 또는 `CLAUDE.local.md`(사무)로 복사하고 `{{ID}}` `{{WT}}` `{{PORT}}` `{{SCOPE}}` 등을 치환한다.
- 공통 규칙은 루트 `CLAUDE.md`(존댓말·두괄식·경계·금지 사항·설계 규율 8절·레인 병행 원칙 9절)이며 모든 worktree가 상위 디렉터리로 상속한다.
- 같은 세션 안에서 쓰는 서브에이전트는 `~/.claude/agents/`의 `fleet-reviewer` `fleet-qa` `fleet-integrator` 셋뿐이고 model 지정이 없어 부모 세션 모델(Fable)을 상속한다.

## 2. 동작 방식

### 2.1 기본 단위

작업 1건 = 카드 1장(`.fleet/tasks/<ID>.md`) = worktree 1개(`wt/<ID>/`) = 브랜치 1개 = 세션 1개. 세션은 worktree를 오가지 않고, 메인 저장소 `C:\work\seohwa`는 읽기 전용 기준점이다. 경계는 `guard-path.sh` PreToolUse 훅이 지킨다.

### 2.2 수동 흐름 (세션 7개를 탭으로 띄우는 방식)

```
fe/be:   task-new → wt-new → (DEV 세션) → verify → review-req → (REVIEWER 판정) → integrate → [사용자 push]
office:  task-new → wt-new → (OFFICE 세션) → verify → review-req → (REVIEWER 판정) → office-publish → 산출물 폴더
RF:      debt.md → task-new RF-… → wt-new → (DEV, 동작 변경 0건) → verify → review → integrate(조용할 때)
```

오케스트레이터(사용자 탭)가 카드를 만들고 `wt-new.sh`가 인쇄한 명령으로 각 역할 세션을 새 탭에서 연다. 개발 세션이 `verify.sh` 통과 후 `review-req.sh`를 부르면 리뷰어 세션이 `판정: approved / changes-requested`를 리뷰 문서 마지막 줄에 쓰고, 인테그레이터가 `integrate.sh` / `office-publish.sh`로 합친다.

### 2.3 자율 운행 루프 (`fleet-loop.sh`)

- 60초 tick마다 보드를 읽고 상태 기계를 돌린다: `assigned` → `agent-run.sh <ID> dev` → `in-progress` → verify 통과 시 `review-req.sh` → `agent-run.sh <ID> reviewer` → 판정.
- `changes-requested`면 개발 재기동, `verify-failed`는 환경 원인이면 자동 복구 후 재검증, 아니면 재기동(최대 `LOOP_MAX_TRIES=3`).
- 티어 A(사무 전부, fe/be 소규모 변경)는 자동 통합·push, 티어 B(스키마·i18n·메뉴·마이그레이션·정책 룰)와 티어 C(RF)는 `awaiting-human`으로 대기.
- 빈 슬롯이 있으면 의존이 충족된 카드를 골라 `wt-new.sh`로 착수시킨다.
- 자유 문장 지시는 `fleet-tell.sh` → `.fleet/inbox/` → `fleet-orch.sh`가 오케스트레이터 에이전트를 헤드리스로 띄워 카드 작성·우선순위·ADR 승인만 처리한다.

### 2.4 에이전트 실행 명령 (모델이 Fable로 고정되는 지점)

```bash
# agent-run.sh — dev / reviewer 공통
( cd "$CWD" && timeout "$TIMEOUT" "$CLAUDE" -p "$P" --permission-mode bypassPermissions )

# fleet-orch.sh — 오케스트레이터
( cd "$FLEET" && timeout "${ORCH_TIMEOUT:-1800}" "$CLAUDE" -p "$P" --permission-mode bypassPermissions )
```

두 곳 모두 `--model`이 없다. `wt-new.sh`가 만드는 `.claude/settings.local.json`에도 `model` 키가 없다. 따라서 대화형 탭 세션과 헤드리스 세션 모두 CLI 기본 모델(Fable)로 돈다.

## 3. 토큰 관점의 문제점

| 작업 종류 | 실제로 하는 일 | 필요한 모델 수준 | 현재 |
| --- | --- | --- | --- |
| 판단·배정·ADR 승인 | 보드·카드·부채 읽고 결정 | 상위 | Fable (적절) |
| 코드 구현·수정 | 파일 읽기·편집·커밋 | 중상위 | Fable (과다) |
| 파일 탐색·grep·심볼 찾기 | 정해진 명령 실행 후 결과 정리 | 하위 | Fable (과다) |
| verify 로그·diff·테스트 결과 읽기 | 긴 출력에서 실패 항목 추출 | 하위 | Fable (과다) |
| 문서 자료 검색·발췌 | 볼트·입력 폴더에서 관련 부분 추리기 | 하위 | Fable (과다) |
| 리뷰 판정 | diff 전체 읽고 체크리스트 판정 | 중상위 | Fable (과다) |

- 개발 에이전트가 탐색·검증 로그 읽기까지 직접 하므로 세션 컨텍스트가 커지고, 커진 컨텍스트가 매 턴 최상위 단가로 재입력된다.
- 리뷰어가 diff 전체를 Fable로 읽는다.
- `AGENT_TIMEOUT=3600`으로 한 카드에 최대 1시간을 Fable이 점유한다.

## 4. 관련 파일

- 공통 규칙: `CLAUDE.md` · 운영 안내: `README.md`
- 역할: `roles/orchestrator.md` `dev-fe.md` `dev-be.md` `office.md` `designer.md` `reviewer.md` `qa.md` `integrator.md`
- 실행: `scripts/agent-run.sh` `fleet-loop.sh` `fleet-orch.sh` `wt-new.sh` · 설정: `.fleet/config.env`
- 서브에이전트: `~/.claude/agents/fleet-reviewer.md` `fleet-qa.md` `fleet-integrator.md` (WSL 홈)

관련 문서: [[멀티에이전트 오케스트레이션 구성 v2(토큰 절약 버전)]]
