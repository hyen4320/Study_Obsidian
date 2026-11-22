# 멀티에이전트 오케스트레이션 구축 가이드 (배포용 v4 · WSL ext4 · 장기기억 DB · 커밋 계정 통일)

작성일: 2026-09-16 (v3: 2026-09-15) · 원본 환경: 클라우드에어 「서화」 개발 관제(`/home/<사용자>/work/seohwa-agents`) · 대상 독자: **Claude Code 로 같은 구조를 자기 프로젝트에 구축하려는 사람**
v2(2026-09-11) 대비 바뀐 것: **작업 경로를 `/mnt/c` 에서 WSL 네이티브 ext4 로 옮겼습니다.** 구조·모델 배분·서브에이전트는 v2 그대로입니다.
**v4(2026-09-16) 에서 추가된 것 두 가지**
1. **커밋 계정 통일** — 사람·에이전트·헤드리스 세션이 만드는 모든 커밋의 작성자·커미터를 `.fleet/config.env` 의 `COMMIT_AUTHOR_NAME` · `COMMIT_AUTHOR_EMAIL` 하나로 고정하고, 리뷰·통합 단계에서 어긋난 커밋을 거부합니다(0-1 (2) · 2-0 (8)).
2. **장기기억 DB 연동(선택)** — 에이전트들이 하나의 Postgres(+pgvector) 를 공용 장기기억으로 읽고 쓰게 합니다. 세션이 끊기거나 인계돼도 결정·사실·작업 기록이 이어집니다(0-1 (8) · 2-7 · 부록 D).
v4 추가분은 **설계 단계이며 서화 환경 실측 전**입니다. v3 의 실측 수치(ext4 · 메모리)는 그대로 유효합니다.

2026-09-15 갱신(v3): **아무것도 없는 상태에서 순서대로 따라가면 끝나도록** 0-0 사전 준비(하드웨어·WSL·도구)와 운영 중 실제로 겪은 장애 5종의 복구 절차를 넣었습니다.

**전체 순서와 소요 시간** — 0-0 사전 준비(30분~2시간, 도구 설치 포함) → 0-1 설정 시트 작성(10분) → 2-0 `fleet-init.sh`(지시문 ⓪, 20분) → 2-1~2-5 스크립트·역할·서브에이전트 생성(지시문 ①~⑤, 2~3시간) → 2-6 첫 시험(30분) → (선택) 2-7 장기기억 DB(지시문 ⑦, 1~2시간). 사람이 직접 타이핑하는 것은 0-0 의 설치 명령과 0-1 의 값 몇 개뿐이고, 나머지는 지시문을 Claude Code 에 붙여 넣으면 Claude 가 만듭니다.

## 결론

이 가이드대로 하면 **판단은 Fable, 구현은 Opus, 정해진 명령·잡일은 Haiku** 로 나뉜 worktree 기반 멀티에이전트 관제를 반나절 안에 세울 수 있습니다. 사람이 손으로 짜는 것은 거의 없습니다. 각 단계의 "Claude에게 줄 지시문" 을 Claude Code 에 그대로 붙여 넣으면 Claude 가 스크립트·역할 파일·서브에이전트를 만듭니다.

v3 에서 바뀐 핵심(v4 에서도 그대로)은 **저장소를 전부 `/home/<사용자>/work/` (ext4) 에 두는 것** 입니다. 서화 환경에서 같은 카드·같은 커밋으로 실측한 결과 프론트 verify 가 **4분 16초에서 20.6초로 12.4배** 빨라졌고, `/mnt/c` 에서 상시 발생하던 Turbopack 빌드 실패(→ webpack 재빌드)가 사라졌습니다. 에이전트를 여러 개 돌리는 구조에서는 이 차이가 그대로 대기 시간과 토큰(재시도)으로 환산되므로, **`/mnt/c` 에 구축하지 마십시오.** 이미 `/mnt/c` 에 구축했다면 부록 A 의 이전 절차를 쓰면 됩니다(서화에서 29분 만에 완료, 검증 11항목 전부 통과).

그리고 에이전트를 돌리기 전에 **2-0 최초 세팅(보호 브랜치·기준선 태그·exclude·경계 훅·커밋 계정)** 을 반드시 먼저 끝냅니다. 이것이 없으면 에이전트가 메인 브랜치를 건드리고 되돌릴 방법이 없습니다.

v4 는 여기에 두 가지를 더합니다. **커밋 계정은 설정 파일 한 곳에서만 정하고 스크립트가 강제합니다** — 세션마다 `git config` 가 다르거나 도구가 붙이는 공동 작성자 줄 때문에 이력이 여러 계정으로 흩어지는 것을 막습니다. **장기기억 DB 는 정본이 아니라 검색용 기억 창고입니다** — ADR·카드·보드 파일이 여전히 정본이고, DB 는 "지난번에 뭘 정했고 뭘 겪었는지" 를 다음 세션이 빠르게 꺼내 보게 합니다.

**가장 흔한 실패는 성능이 아니라 메모리입니다.** 이 구조는 worktree 하나마다 세션 1개가 붙고 백엔드 worktree 는 컨테이너를 2개씩 띄웁니다. WSL 기본 메모리(호스트의 50%)에서 헤드리스 세션 6개와 컨테이너 11개를 동시에 돌렸다가 스왑까지 소진돼 **6개가 한꺼번에 강제 종료된 실측 사례**가 있습니다(2026-09-15). 0-0 에서 메모리를 먼저 정하고, 동시 상한을 그 값에서 산정하십시오.

* * *

## 0-A. 이 문서를 쓰는 법 (새 기계에서 시작한다면 여기부터 읽으십시오)

결론: **이 문서를 통째로 Claude 에 붙여 넣지 마십시오.** 순서는 셋입니다 — ① 0-0 은 사람이 셸에서 직접 하고, ② 이 파일을 새 기계에 복사한 뒤, ③ Claude Code 를 열어 아래 킥오프 프롬프트 하나만 붙여 넣습니다. 그 다음부터는 Claude 가 이 문서의 지시문 ⓪~⑥ 을 순서대로 실행하고, 단계마다 사람의 확인을 받습니다.

### (1) 파일을 새 기계로 옮깁니다

Obsidian 동기화·git·USB 어느 쪽이든 됩니다. WSL 안의 아무 경로에 두면 되고, 관제 저장소를 만들 폴더 옆이 편합니다.

```bash
mkdir -p ~/work && cp /mnt/c/<받은 경로>/멀티에이전트-오케스트레이션_구축_배포용_v4\(WSL-ext4\).md ~/work/
```

### (2) 0-0 사전 준비는 사람이 합니다

WSL 설치 · `.wslconfig` · 도구 설치 · `claude` 로그인과 모델 ID 확인까지는 Claude 가 대신할 수 없습니다(설치 중 비밀번호·로그인·재부팅이 끼어 있습니다). 0-0 (5) 체크리스트 6줄이 전부 통과하면 다음으로 갑니다.

### (3) 관제 저장소 뼈대만 만들고 Claude 를 엽니다

```bash
mkdir -p ~/work/<프로젝트>-agents && cd ~/work/<프로젝트>-agents && git init
claude --model claude-fable-5-1        # Fable 이 없으면 --model opus
```

### (4) 킥오프 프롬프트 — 이것만 붙여 넣습니다

```
~/work/멀티에이전트-오케스트레이션_구축_배포용_v4(WSL-ext4).md 를 읽으십시오.
저는 이 기계에 <프로젝트> 용 멀티에이전트 관제를 처음부터 구축하려 합니다.

- 코드 저장소: /home/<사용자>/work/<프로젝트>   (프론트: <있음/없음, 폴더명> · 백엔드: <있음/없음, 폴더명>)
- 관제 저장소(지금 이 디렉터리): /home/<사용자>/work/<프로젝트>-agents
- 개발 브랜치: <브랜치명>   · 원격: <origin 있음/없음>
- 이 기계: RAM <N>GB · 코어 <M>개 · WSL 메모리 <free -h 의 total>
- 사무(문서) 레인: <씀/안 씀>   · Obsidian 볼트: <경로 또는 없음>
- 커밋 계정: <이름> <이메일>   (모든 에이전트 커밋이 이 계정으로 남습니다)
- 장기기억 DB: <씀/안 씀>

진행 방식:
1. 먼저 0-0 (5) 체크리스트를 실제 명령으로 점검해 부족한 것을 알려 주십시오. 부족하면 거기서 멈추고 무엇을 설치해야 하는지만 알려 주십시오.
2. 통과하면 0-0 (1) 산정식으로 MAX_WT_FE · MAX_WT_BE · MAX_CONCURRENT_AGENTS 값을 계산해 제안하십시오.
3. 그 뒤 2-0 지시문 ⓪ 부터 2-6 지시문 ⑥ 까지 순서대로 실행하고, 장기기억 DB 를 쓴다면 2-7 지시문 ⑦ 까지 이어가십시오.
4. 지시문 하나가 끝날 때마다 무엇을 만들었고 무엇이 실패했는지 두괄식으로 보고하고, 제 확인을 받은 뒤 다음으로 넘어가십시오.
5. 모르는 값(브랜치·포트·저장소 경로·모델 ID)은 지어내지 말고 물어보십시오. 물을 때는 0-1 의 「질문 화면 규격」대로 — 이 값이 무엇인지, 왜 필요한지, 어떻게 확인하는지, 예시와 기본값 — 을 함께 보여 주십시오. 문서에 없는 구조를 임의로 추가하지 마십시오.
6. 빈칸이 <...> 그대로 남아 있으면 제가 모르는 값이라는 뜻입니다. 그 값만 골라 한 번에 하나씩, 확인 명령과 함께 물어보십시오.
```

#### 빈칸 채우는 법 — 모르면 비워 두고 붙여 넣어도 됩니다

결론: **모르는 칸은 `<...>` 그대로 두십시오.** Claude 가 확인 명령을 알려 주며 하나씩 묻습니다. 아래 표는 미리 채우고 싶을 때 쓰는 안내입니다.

| 빈칸 | 무엇인가 | 확인하는 법 | 예시 | 모르면 |
| --- | --- | --- | --- | --- |
| 코드 저장소 | 에이전트가 고칠 원래 프로젝트 폴더 | `ls ~/work` · 폴더 안에 `.git` 이 있는지 `ls -a` | `/home/com/work/seohwa` | 비워 두면 Claude 가 `~/work` 아래 git 폴더 목록을 보여 줍니다 |
| 프론트·백엔드 폴더 | 코드 저장소 안의 하위 폴더 이름 | `ls <코드 저장소>` — `package.json` 이 있는 쪽이 프론트, `requirements.txt`·`docker-compose.yml` 이 있는 쪽이 백엔드 | `frontend` · `backend` | 비워 두면 자동 감지 |
| 관제 저장소 | 이 가이드가 새로 만드는 폴더(지금 Claude 를 연 곳) | `pwd` | `/home/com/work/seohwa-agents` | 지금 폴더로 잡습니다 |
| 개발 브랜치 | 사람이 평소 작업하는 브랜치. 에이전트는 여기에 직접 커밋하지 않음 | `git -C <코드 저장소> branch --show-current` | `dev-ryu` | 현재 브랜치로 제안합니다 |
| 원격 | push 할 곳이 있는지 | `git -C <코드 저장소> remote -v` — 줄이 나오면 있음 | `origin 있음` | 자동 감지 |
| RAM · 코어 | 윈도우 PC 사양 | 작업 관리자 → 성능 탭 | `64GB · 16개` | 비워 두면 WSL 에서 보이는 값만 씁니다 |
| WSL 메모리 | WSL 이 실제로 쓸 수 있는 메모리 — **동시에 돌릴 에이전트 수를 정합니다** | WSL 에서 `free -h` 의 `Mem:` 줄 `total` | `31Gi` | Claude 가 직접 확인합니다 |
| 사무(문서) 레인 | 보고서·공문 같은 문서도 에이전트에게 맡길지 | — | `씀` | `안 씀` 으로 시작해도 나중에 켤 수 있습니다 |
| Obsidian 볼트 | 에이전트가 **읽기만** 할 메모 폴더 | 윈도우 경로 `C:\Users\<이름>\Obsidian\업무` → WSL 에서는 `/mnt/c/Users/<이름>/Obsidian/업무` | `/mnt/c/Users/com/Obsidian/업무` | `없음` |
| 커밋 계정 | 에이전트가 만드는 모든 커밋에 찍힐 이름·이메일 | `git config user.name` · `git config user.email` / GitLab·GitHub 프로필의 이메일 | `류현 ryu@example.com` | 현재 git 설정을 보여 주고 확인받습니다. **이메일이 원격 계정과 다르면 원격 화면에서 다른 사람으로 보입니다** |
| 장기기억 DB | 에이전트끼리 기억을 공유하는 DB 를 둘지 | — | `안 씀` | `안 씀` 을 권합니다. 구축이 끝난 뒤 2-7 로 켤 수 있습니다 |

### (5) 그 다음

Claude 가 `fleet-init.sh` 를 만들고 실행하면 0단계에서 설정 시트(0-1)를 **질문으로** 채웁니다. 감지되는 값은 Enter 로 넘기고, `BASELINE_COMMIT` · `PROTECTED_BRANCHES` · `HOTSPOTS` · `COMMIT_AUTHOR_NAME`/`COMMIT_AUTHOR_EMAIL` 만 눈으로 확인하십시오. 마지막 2-6 첫 시험까지 통과하면 구축이 끝납니다.

지시문 ①~⑦ 은 이 문서 2절에 코드블록으로 있습니다. Claude 가 문서를 읽으므로 사람이 다시 붙여 넣을 필요는 없지만, 세션이 끊겨 처음부터 다시 할 때는 해당 코드블록만 복사해 이어가면 됩니다.

* * *

## 0-0. 사전 준비 (아무것도 없는 상태에서 시작한다면 여기부터)

결론: **메모리를 먼저 정하고, 도구를 깔고, 모델이 뜨는지 확인한 뒤에** 구축을 시작합니다. 이 절을 건너뛰면 2-6 첫 시험이나 운영 첫날에 반드시 되돌아오게 됩니다.

### (1) 하드웨어 — 메모리가 동시 상한을 정합니다

이 구조의 메모리 소비는 단순합니다.

| 항목 | 대략 |
| --- | --- |
| 헤드리스 Claude 세션 1개 | 0.3~1.0 GB |
| be worktree 1개 (앱 + Postgres 컨테이너 2개) | 1.0~1.5 GB |
| fe worktree 1개 (`next dev` 또는 `next build`) | 0.5~1.5 GB |
| 통합 검증 스택(`wt/integration`) | 1.0~1.5 GB |
| 로컬 확인 스택(사용자가 화면을 보는 용도) | 1.0~1.5 GB |
| (v4 · 선택) 장기기억 DB — Postgres 1개 + 임베딩 모델(로컬) | 0.3~0.5 GB + 0.5~1.0 GB (임베딩을 끄면 앞쪽만) |

**산정식**: `동시 카드 수 ≈ (WSL 메모리 − 3GB − 장기기억 DB 몫) ÷ 2GB`. 장기기억 DB 를 안 쓰면 그 몫은 0 입니다. 8GB 면 동시 2장, 12GB 면 4장, 16GB 면 6장이 상한이고, 여기에 여유 1장을 빼고 시작하는 편이 안전합니다.

**메모리가 넉넉한 기계(24GB 이상)라면 이 절은 빠르게 넘어가도 됩니다.** `MAX_WT_FE`·`MAX_WT_BE` 를 3/3 으로, `MAX_CONCURRENT_AGENTS` 를 4~6 으로 시작하십시오. 대신 병목이 메모리에서 아래로 옮겨갑니다.

| 다음 병목 | 증상 | 대응 |
| --- | --- | --- |
| **CPU 코어** | `next build` 와 `pytest` 가 동시에 여러 개 돌면 각각이 느려집니다. 코어 수보다 많은 빌드를 동시에 돌리면 전체 처리량이 오히려 떨어집니다 | 동시 카드 수를 코어 수의 절반 이하로. `.wslconfig` 의 `processors` 도 함께 올립니다 |
| **디스크** | worktree 마다 `node_modules` 실물이 생깁니다(프로젝트에 따라 수백 MB~1GB) | 저장소 크기의 4~5배를 확보. `df -h /home` 을 주기적으로 확인 |
| **모델 사용 한도** | 메모리와 무관하게 세션이 중단됩니다. 특히 Fable 은 한도가 빨리 찹니다(2026-09-14 실제 발생 — RF 카드가 한도 소진으로 중단) | `*-hard` 호출 비율을 카드의 30% 이하로 유지. 한도에 걸리면 `MODEL_HARD` 를 `opus` 로 내리고 카드에 `handover:` 로 기록 |
| **`claude -p` 의 600초 백그라운드 한계** | 검증 결과 없이 세션이 끝납니다 | 메모리와 무관한 제약입니다. 2-3 규격 6번대로 검증을 포그라운드에서 돌립니다 |
| **도커 데몬** | 컨테이너가 10개를 넘으면 기동·정지가 눈에 띄게 느려집니다 | be 동시 카드를 3 이하로. 끝난 카드는 `wt-rm.sh` 로 즉시 정리(컨테이너까지 내려갑니다) |

WSL 메모리는 기본이 호스트의 50% 입니다. 늘리려면 윈도우의 `C:\Users\<사용자>\.wslconfig` 에 적고 PowerShell 에서 `wsl --shutdown` 후 다시 엽니다.

```ini
[wsl2]
memory=12GB
swap=4GB
processors=8
```

확인: WSL 에서 `free -h` 의 total 이 지정한 값인지 봅니다. **이 값이 0-1 설정 시트의 `MAX_WT_FE` · `MAX_WT_BE` 와 동시에 띄울 헤드리스 세션 수를 결정합니다.**

디스크는 저장소 크기의 4~5배를 잡습니다(worktree 마다 `node_modules` 실물이 생길 수 있습니다). `/home` 여유는 `df -h /home` 으로 확인합니다.

### (2) WSL 과 기본 도구

```bash
# 윈도우 PowerShell (관리자)
wsl --install -d Ubuntu-24.04
wsl -l -q                     # 배포판 등록명 확인 — UNC 경로에 그대로 씁니다

# WSL 셸
sudo apt-get update
sudo apt-get install -y git curl build-essential python3 python3-venv python3-pip pandoc fonts-nanum
node -v                       # 없으면 nvm 으로 설치 (아래)
```

Node 는 배포판 패키지 대신 nvm 을 권합니다(Next 빌드가 버전을 탑니다).

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
exec $SHELL -l && nvm install --lts && node -v && npm -v
```

Docker 는 두 가지 중 하나입니다 — **Docker Desktop(윈도우) + WSL 통합**을 쓰면 `/usr/bin/docker` 가 `/mnt/wsl/docker-desktop/...` 심볼릭 링크가 되어 **Desktop 이 꺼지면 docker 명령 자체가 사라집니다**(be verify 가 1단계에서 멈추는 흔한 원인). WSL 안에 직접 `docker.io` 를 깔면 그 문제는 없지만 윈도우 쪽에서 컨테이너가 보이지 않습니다. 어느 쪽이든 `docker ps` 가 되는 것을 확인하고 넘어갑니다.

### (3) Claude Code 와 모델

```bash
npm i -g @anthropic-ai/claude-code
claude --version
claude                        # 최초 1회 로그인
```

세션 안에서 `/model` 로 **자기 계정에서 실제로 뜨는 모델 ID** 를 확인하고, 쓰려는 것 하나하나를 단독 기동해 봅니다.

```bash
claude --model claude-fable-5-1 -p "hi"     # 판단용. 안 되면 MODEL_ORCH·MODEL_HARD 를 opus 로
claude --model opus -p "hi"                 # 구현용
claude --model haiku -p "hi"                # 잡일용
```

Fable 이 없어도 구조는 그대로 돌아갑니다(절감의 대부분은 유지됩니다). **여기서 뜨지 않는 ID 를 `config.env` 에 적으면 모든 헤드리스 기동이 실패합니다.**

### (4) 저장소 배치

코드 저장소가 아직 `/mnt/c` 에 있다면 먼저 옮깁니다. worktree 를 아직 만들지 않은 상태라면 복사 한 번이면 됩니다.

```bash
mkdir -p ~/work
rsync -a /mnt/c/work/<프로젝트>/ ~/work/<프로젝트>/
git -C ~/work/<프로젝트> status          # 브랜치·변경이 그대로인지
git -C ~/work/<프로젝트> log --oneline -3
```

이미 worktree 가 딸린 구성을 통째로 옮기는 경우는 **부록 A** 를 그대로 따릅니다.

### (5) 사전 준비 체크리스트

- [ ] `free -h` 의 total 이 의도한 값 (`.wslconfig` 반영 확인)
- [ ] `df -h /home` 여유가 저장소의 4~5배
- [ ] `git` `node` `npm` `python3` `pandoc` `docker ps` 전부 동작
- [ ] `claude --version` · `/model` 로 확인한 ID 3개가 `-p "hi"` 로 각각 기동됨
- [ ] 코드 저장소가 `/home/<사용자>/work/` 아래에 있고 `git status` 가 깨끗함
- [ ] 커밋에 쓸 계정(이름·이메일)을 정함 — 원격(GitLab·GitHub)에 등록된 이메일과 같아야 원격 화면에서 사용자로 연결됩니다
- [ ] (사무 레인을 쓴다면) Obsidian 볼트 경로와 산출물함으로 쓸 윈도우 폴더를 정함

* * *

## 0. 전제 환경

| 항목          | 원본 환경                                                        | 비고                                                                            |
| ----------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| OS          | Windows 11 + WSL2(Ubuntu 24.04)                              | 작업은 전부 WSL 셸에서                                                                |
| **작업 경로**   | **`/home/<사용자>/work/` (ext4)**                               | **`/mnt/c` 에 두지 않습니다.** 0.1 참조                                                |
| Claude Code | WSL 안에 설치, `claude` 명령 사용 가능                                 | `claude --version`. Fable 모델이 되는 플랜                                           |
| 코드 저장소      | `/home/<사용자>/work/<프로젝트>` (git)                              | 프론트 `frontend/`(Next.js) · 백엔드 `backend/`(FastAPI + Postgres, docker compose) |
| 관제 저장소      | `/home/<사용자>/work/<프로젝트>-agents` (이 가이드가 만드는 것)              | 코드 저장소와 **분리**                                                                |
| 사무 저장소      | `/home/<사용자>/work/<프로젝트>-office` · `/home/<사용자>/work/office` | 문서 산출물용, git                                                                  |
| 메모          | Obsidian 볼트 `/mnt/c/Users/<이름>/Obsidian/업무`                  | **윈도우에 그대로 둡니다.** 에이전트는 읽기만                                                   |
| 산출물함        | `C:\work\산출물\`                                               | 발행된 docx·pdf·pptx·xlsx **사본**. 한컴·엑셀로 여는 유일한 윈도우 경로                           |
| 도구          | git · node · npm · python3 · docker · pandoc · marp(선택)      | WSL 안에 설치                                                                     |
| 브랜치         | 기준 브랜치 `integration` (없으면 초기화 스크립트가 만듦)                      | 원격 push 는 사람이 직접                                                              |

### 0.1 왜 ext4 인가 (실측)

같은 기계에서 파일 1000개를 생성·읽기·삭제한 값입니다.

| 경로 | 생성 | 읽기 | 삭제 |
| --- | --- | --- | --- |
| `/mnt/c` (9p) | 1.90초 | 1.13초 | 0.32초 |
| `/home` (ext4) | 0.01초 | 0.01초 | 0.00초 |

생성 기준 190배입니다. 실제 파이프라인에 미친 영향(서화, 같은 카드·같은 커밋 `TASK-CPREPORT-001 @ 54467c5`):

| 항목 | `/mnt/c` | ext4 |
| --- | --- | --- |
| fe verify 1회 | 4분 16초 (Turbopack 실패 → webpack 재빌드) | **20.6초** (`.next` 삭제 후 클린, Turbopack) |
| be worktree 생성 | 36초 | 2.4초 |
| be verify 1회 | — | 50.2초 |

`/mnt/c` 는 `metadata` 마운트 옵션이 없으면 `chmod`·`utime` 이 실패해 **Next 16 Turbopack 이 `.next/` 쓰기에서 죽습니다.** ext4 에서는 이 회피 코드 자체가 필요 없습니다. venv 를 리눅스 FS 에 따로 만들고 심볼릭 링크로 거는 우회, `dubious ownership` 때문에 저장소마다 `safe.directory` 를 등록·해제하던 처리, 사무 저장소를 리눅스 FS 에서 `git init` 해 `.git` 을 복사하던 처리도 전부 사라집니다.

### 0.2 윈도우와 만나는 지점은 세 곳뿐입니다

| 지점 | 방향 | 어떻게 |
| --- | --- | --- |
| Obsidian 볼트 | WSL → 윈도우 **읽기만** | `VAULT=/mnt/c/Users/<이름>/Obsidian/업무` |
| 산출물함 | WSL → 윈도우 **쓰기(사본)** | `office-publish.sh` 가 `OUTBOX/<프로젝트>/<ID>/` 로 `cp`. 정본은 저장소 안 `산출물/<ID>/` |
| PDF·PPTX 변환 | WSL → 윈도우 Chrome 실행 | `wslpath -w` 가 `/home/...` 을 `\\wsl.localhost\<배포판>\home\...` UNC 로 바꿔 넘깁니다 |

세 번째가 v2 에서 유일하게 걸림돌로 지목됐던 부분입니다. **실측 결과 UNC 경유로 동일하게 동작합니다** — 같은 문서로 만든 PDF 가 `/mnt/c` 산출물과 바이트 수까지 같았고(21,213 바이트), 한글도 정상, 소요도 0.6초로 같았습니다. 변환 스크립트에서 `[[ "$OUT" == /mnt/c/* ]]` 같은 **경로 조건을 넣지 마십시오**(넣으면 조용히 저품질 폴백으로 떨어집니다).

윈도우 탐색기에서 작업 파일을 볼 때의 경로는 `\\wsl.localhost\<배포판 등록명>\home\<사용자>\work\...` 입니다. 등록명은 `wsl -l -q` 로 확인합니다(`Ubuntu` 가 아니라 `Ubuntu-24.04` 인 경우가 많습니다).

* * *

## 0-1. 사람마다 다른 값 — 설정 시트 (구축 전에 채운다)

결론: 아래 표의 "내 값" 열을 먼저 채우고 시작합니다. 전부 `.fleet/config.env` 한 파일에 들어가며, 스크립트와 역할 파일은 이 값을 읽습니다. **"없으면" 열이 "넘어감" 인 항목은 비워 두면 스크립트가 그 단계를 건너뛰고, "필수" 인 항목은 비어 있으면 `fleet-init.sh` 가 멈춥니다.** 원본(서화)에서 스크립트에 박혀 있던 값(브랜치명, 청결 검사 예외, 핫스팟 4종, 커밋 접두사 등)은 배포판에서는 전부 변수로 뺍니다 — 지시문 ①·② 에 그렇게 시키는 문장이 들어 있습니다.

### (1) 경로 · 저장소

| 변수 | 뜻 | 원본(서화) 값 | 없으면 |
| --- | --- | --- | --- |
| `REPO` | 코드 저장소(읽기 전용 기준점) | `/home/com/work/seohwa` | 필수 |
| `FLEET` | 관제 저장소(이 가이드가 만드는 곳) | `/home/com/work/seohwa-agents` | 필수 |
| `OFFICE_SEOHWA` | 프로젝트 사무 저장소(`OF-` 카드) | `/home/com/work/seohwa-office` | 비우면 사무 레인 자체를 넘어감 |
| `OFFICE_GENERAL` | 일반 사무 저장소(`GO-` 카드) | `/home/com/work/office` | 비우면 `GO-` 카드 거부 |
| `VAULT` | 개인 메모(읽기만) | `/mnt/c/Users/com/Obsidian/업무` | 비우면 볼트 차단 규칙만 생략 |
| **`OUTBOX`** | **발행 산출물 사본을 두는 윈도우 폴더** | `/mnt/c/work/산출물` | 비우면 복사 단계를 넘어감(정본은 저장소 안) |
| `HOST_IP` | WSL 에서 보이는 PC IP(`NEXTAUTH_URL`, 컨테이너 `FRONTEND_ORIGIN`) | `192.168.0.163` | 필수(`hostname -I` 첫 값) |
| `VENV_REAL` | 사무 변환용 venv 경로 | `~/.local/share/<프로젝트>-fleet/venv` | 기본값 사용. **ext4 에서는 `$FLEET/.venv` 에 직접 만들어도 됩니다** |

`REPO` `FLEET` `OFFICE_*` 는 **전부 `/home/` 아래여야 합니다.** `fleet-init.sh` 가 `/mnt/` 로 시작하면 경고하고 확인을 받습니다(0.1 의 실측 차이 때문입니다). `VAULT` 와 `OUTBOX` 만 `/mnt/c` 입니다.

### (2) 브랜치 · 커밋 경계

| 변수 | 뜻 | 원본(서화) 값 | 없으면 |
| --- | --- | --- | --- |
| `DEV_BRANCH` | 사람이 쓰는 개발 브랜치. `integration` 을 여기서 분기하고, 여기에는 에이전트가 직접 커밋하지 않음 | `dev-ryu` | 필수 |
| `BASE_BRANCH` | 에이전트 통합 지점(유일) | `integration` | 기본값 사용 |
| `BASELINE_COMMIT` | **이 커밋 이전은 건드리지 않는다** — `integration` 을 만들 기준점이자 되돌리기 하한. 태그 `fleet-baseline-*` 가 여기에 찍힘 | 비움(=`DEV_BRANCH` HEAD) | 비우면 현재 HEAD |
| `PROTECTED_BRANCHES` | 에이전트가 checkout·commit·merge 하면 안 되는 브랜치 목록(쉼표) | `dev-ryu,development,main,master` | 비우면 `DEV_BRANCH` 만 보호 |
| `REMOTE_NAME` | push 대상 원격 이름 | `origin` | 원격 없으면 push 단계 넘어감 |
| `LOOP_AUTO_PUSH` | 통합 성공 시 자동 push | **배포판 기본 0** | 0 이면 사람이 push |
| `DIRTY_EXCEPT` | `fleet-init` 워킹트리 청결 검사에서 무시할 경로(쉼표) | 커밋 안 된 실험 폴더 2개 | 비우면 예외 없음 |
| `NO_TOUCH_PATHS` | 에이전트 수정 금지 경로(쉼표). 범위 검사·`guard-path` 가 봄 | `backend/data/,backend/terraform/` | 비우면 카드 `scope` 만 적용 |
| `NO_TOUCH_FILES` | 에이전트 수정 금지 파일(쉼표) | `package.json,package-lock.json,requirements.txt,docker-compose.yml,.gitignore,.gitattributes` | 기본값 사용 |
| **`COMMIT_AUTHOR_NAME`** | **(v4) 모든 커밋의 작성자·커미터 이름** | 사람 계정 이름 또는 전용 봇 계정 이름 | **필수** |
| **`COMMIT_AUTHOR_EMAIL`** | **(v4) 모든 커밋의 작성자·커미터 이메일** — 원격에 등록된 이메일 | `<계정>@<도메인>` | **필수** |
| `COMMIT_COAUTHOR` | (v4) 커밋 메시지의 `Co-Authored-By:` 같은 공동 작성자 줄 허용 여부 | 0 | 0 이면 붙지 않게 막고, 붙은 커밋은 리뷰에서 거부 |
| `COMMIT_IDENTITY_STRICT` | (v4) 계정이 다른 커밋이 섞이면 `review-req.sh`·`integrate.sh` 가 거부 | 1 | 0 이면 경고만 |

### (3) 프로젝트 특성 (스택에 따라 넘어가는 것이 많다)

| 변수 | 뜻 | 원본(서화) 값 | 없으면 |
| --- | --- | --- | --- |
| `FE_DIR` / `BE_DIR` | 프론트·백엔드 하위 폴더 | `frontend` / `backend` | 한쪽이 비면 그 레인 전체를 넘어감 |
| `FE_VERIFY` | 프론트 검증 명령 순서 | `tsc,lint,i18n,build` | 비우면 `tsc,lint` 만 |
| `FE_I18N_DIRS` | ko/en 키 짝 검사 대상 | `frontend/messages` | 비우면 i18n 검사 넘어감 |
| `FE_BUILD_FALLBACK` | Turbopack 실패 시 대체 빌드 | **ext4 에서는 비움** | 비우면 재시도 없음 — 0.1 참조 |
| `LINT_BASELINE` | 기존 lint 오류 수를 기준선으로 인정(변경 파일만 엄격) | 1 (기준선 286건) | 0 이면 전체 0건 요구 |
| `BE_COMPOSE_SERVICE` | 백엔드 컨테이너 서비스명 | `seohwa` | 필수(be 레인 사용 시) |
| `BE_VERIFY` | 백엔드 검증 순서 | `import,pytest,restart,openapi,upgrade` | 비우면 `pytest` 만 |
| `USE_ALEMBIC` | 스키마 마이그레이션 사용 여부(락 `alembic` · `heads` 1개 검사) | 1 | 0 이면 스키마 락·검사 넘어감 |
| `OPENAPI_PATH` | be RF 카드의 "동작 변경 0건" 판정 기준 파일 | `openapi.json` | 비우면 RF 판정을 테스트 결과로만 |
| `HOTSPOTS` | 충돌 핫스팟 락 이름과 경로(`이름=경로;…`) | `menus=frontend/app/lib/menus.ts;messages=frontend/messages;alembic=backend/migrations;templates=office-templates` | 비우면 락 기능 넘어감 |
| `COMMIT_PREFIXES` | 허용 커밋 접두사 | `Feat:,Fix:,Chor:,Refactor:` | 비우면 검사 안 함 |
| `ENV_TEMPLATE` | `backend/.env` 가 없을 때 복사할 원본 | `backend/.env.development` | 비우면 `.env` 없으면 멈춤 |
| `SECRET_KEYS` | 생성해 채우고 절대 출력하지 않을 키 | `CREDENTIAL_KEY` | 비우면 생성 단계 넘어감 |
| `TIER_B_PATTERNS` | 사람 승인이 필요한 변경 경로(정규식) | i18n · 메뉴 · 마이그레이션 · 정책 룰 | 비우면 파일·줄 수 기준만 |

### (4) 포트 · 동시 상한 · 루프

| 변수 | 원본 값 | 비고 |
| --- | --- | --- |
| `FE_PORT_BASE` / `BE_API_PORT_BASE` / `BE_PG_PORT_BASE` | 3101 / 8101 / 5433 | worktree 순번으로 +1. 3000·8000·5432 는 메인 작업본 몫 |
| `INT_FE_PORT` / `INT_API_PORT` / `INT_PG_PORT` | 3100 / 8100 / 5532 | `wt/integration` 전용 |
| `MAX_WT_FE` / `MAX_WT_BE` / `MAX_WT_OFFICE` | 3 / 3 / 3 | **0-0 (1) 산정식으로 정합니다.** be 는 worktree 당 컨테이너 2개라 가장 무겁습니다. WSL 8GB 면 `1 / 1 / 2`, 12GB 면 `2 / 2 / 3`, 16GB 이상에서만 `3 / 3 / 3` |
| `MAX_CONCURRENT_AGENTS` | 2 | **동시에 띄우는 헤드리스 세션 수.** 위 상한과 별개입니다 — worktree 가 5개 있어도 세션은 2개만 돌립니다. WSL 8GB 면 1, 12GB 면 2, 16GB 이상이면 3 |
| `LOOP_TICK` / `LOOP_REPORT_EVERY` | 60 / 3600 (초) |  |
| `LOOP_MAX_TRIES` / `HARD_AFTER_TRIES` | 3 / 3 | `HARD_AFTER_TRIES` ≤ `LOOP_MAX_TRIES` 여야 `*-hard` 가 한 번은 불림 |
| `LOOP_TIER_FILES` / `LOOP_TIER_LINES` | 3 / 200 | 이 이하만 자동 통합(티어 A) |
| `AGENT_TIMEOUT` / `ORCH_TIMEOUT` | 3600 / 1800 (초) | ext4 로 옮기면 verify 가 빨라지므로 줄여도 됩니다 |

### (5) 모델

| 변수 | 원본 값 | 없으면 |
| --- | --- | --- |
| `MODEL_ORCH` / `MODEL_HARD` | `claude-fable-5-1` | 계정에 Fable 이 없으면 `opus` 로 — 절감의 대부분은 유지됨 |
| `MODEL_DEV` / `MODEL_REVIEW` | `opus` |  |
| `MODEL_OFFICE` / `MODEL_DESIGN` | `opus[1m]` | 1M 이 없으면 `opus` |
| `MODEL_HELPER` | `haiku` |  |

실제 사용 가능한 ID 는 `claude` 안에서 `/model` 로 확인하고, `claude --model <ID> -p "hi"` 로 한 번 기동해 봅니다.

### (6) 사무 · 문서 (문서 레인을 안 쓰면 이 표 전체를 넘어감)

| 변수 | 원본 값 | 없으면 |
| --- | --- | --- |
| `OFFICE_KINDS` | `report minutes official email slides sheet research schedule analysis` | 기본값 사용 |
| `OFFICE_ID_PREFIXES` | `OF=seohwa;GO=general` | 접두사가 저장소와 `OUTBOX` 하위 폴더를 결정 |
| `OFFICE_EXPR_RULES` | 프로젝트 특유의 표현 규율(○/× 표현) | 비우면 검사 넘어감 |
| `OFFICE_SECRET_PATTERNS` | 기밀 값 검사 정규식(키·비밀번호·주민번호·계좌) | 기본값 사용 |
| `HWP_INPUT` | hwp 입력을 `hwp2txt` 로 추출 시도 | 0 이면 사용자에게 pdf/docx 변환 요청 |
| `VAULT_GIT` | 볼트를 git 으로 관리 — 원본 0 | 0 이면 볼트 손대지 않음 |

### (7) 작업 ID 규칙

| 변수 | 원본 값 | 비고 |
| --- | --- | --- |
| `ID_PREFIX_FE` / `ID_PREFIX_BE` | `TASK` / `BE` | 프론트 카드는 `frontend/docs/<ID>.md` 지시서와 같은 이름 |
| `ID_PREFIX_RF` | `RF-<fe\|be>` | 리팩토링(동작 변경 0건) |
| `ID_PREFIX_OF` / `ID_PREFIX_GO` | `OF` / `GO` | 사무 |

### (8) 장기기억 DB (v4 · 쓰지 않으면 이 표 전체를 넘어감)

| 변수 | 원본 값(권장 시작값) | 없으면 |
| --- | --- | --- |
| `MEMORY_ENABLE` | 0 → 준비되면 1 | 0 이면 2-7 전체와 관련 프롬프트 문장을 넘어감 |
| `MEMORY_PG_PORT` | 5632 | 메인(5432)·worktree(5433~)·통합(5532)과 겹치지 않게 |
| `MEMORY_DB_URL` | `postgresql://fleet_mem:<생성>@127.0.0.1:5632/fleet_memory` | `SECRET_KEYS` 처럼 생성해 채우고 **출력하지 않음** |
| `MEMORY_EMBED` | `none` → 이후 `local` | `none` 이면 키워드(전문 검색·태그)만. `local` 이면 pgvector 벡터 검색을 함께 씀 |
| `MEMORY_EMBED_MODEL` | `[확인 필요: 한국어 지원·상용 라이선스 되는 로컬 임베딩 모델]` | `MEMORY_EMBED=local` 일 때만 필수 |
| `MEMORY_TOPK` | 8 | 작업 시작 시 불러올 기억 수 |
| `MEMORY_TTL_DAYS` | 90 | 작업 기록(episode)의 기본 유효기간. 사실·결정은 만료 없음 |
| `MEMORY_WRITE_FACT_ROLES` | `orchestrator,integrator` | 사실·결정(fact·decision)을 쓸 수 있는 역할. 나머지는 작업 기록만 |

### 질문 화면 규격 — 친절하게 묻기 (v4)

결론: **`fleet-init.sh` 와 Claude 는 값을 물을 때마다 같은 6줄 형식을 씁니다.** 변수 이름만 던지고 입력을 기다리지 않습니다. 사용자가 이 문서를 읽지 않았어도 화면만 보고 답할 수 있어야 합니다.

```
[3/12] 보호할 브랜치 (PROTECTED_BRANCHES)
  무엇   에이전트가 checkout·커밋·머지하면 안 되는 브랜치 목록입니다.
  왜     여기 적힌 브랜치는 스크립트가 막아 줍니다. 빠뜨리면 에이전트가 그 브랜치를 건드릴 수 있습니다.
  확인   git -C /home/com/work/seohwa branch -a   → 지금 있는 브랜치: dev-ryu, development, main
  예시   dev-ryu,development,main,master   (쉼표로 구분, 띄어쓰기 없이)
  입력 > [Enter = dev-ryu,development,main]  _
```

- **진행 표시** `[현재/전체]` 를 붙입니다. 사무 레인·장기기억을 n 으로 답하면 전체 수가 줄어든 것을 바로 반영합니다.
- **감지값이 있으면 `[Enter = 값]`** 으로 보여 주고, 어디서 감지했는지 한 줄 덧붙입니다(`← git branch --show-current`).
- **확인 줄에는 실제로 실행한 결과**를 보여 줍니다. 사용자가 명령을 따로 칠 필요가 없게 합니다.
- **잘못된 값은 이유와 함께 다시 묻습니다** — 예: "`/mnt/c/...` 는 9p 경로라 느립니다(0.1). `/home/...` 아래 경로를 권합니다. 그래도 쓰시겠습니까 (y/N)". 세 번 틀리면 그 값을 건너뛰고 마지막 요약표에 `⚠ 미입력` 으로 남깁니다(필수값이면 거기서 멈춤).
- **`?` 를 입력하면** 이 문서의 해당 절 번호와 추가 설명 두세 줄을 보여 주고 같은 질문으로 돌아옵니다.
- **비밀 값은 묻지 않고 생성**합니다. 화면에는 `(생성됨 · 표시하지 않음)` 만 찍습니다.
- **마지막 요약표**는 `변수 | 값 | 출처(감지/입력/기본값) | 비고` 네 열로 보여 주고, "이대로 저장할까요 (Y/n) · 고칠 번호 입력" 으로 받습니다. 번호를 넣으면 그 질문만 다시 묻습니다.

#### 질문 목록과 안내 문구

| # | 변수 | 무엇 (화면 문구) | 확인 (스크립트가 실행해 보여 줄 것) | 예시 | 기본값 |
| --- | --- | --- | --- | --- | --- |
| 1 | `REPO` | 에이전트가 고칠 원래 코드 저장소입니다. 에이전트는 여기를 직접 고치지 않고 복제본(worktree)에서 작업합니다 | 상위 폴더의 `.git` 위치 · `git status --short \| wc -l` | `/home/com/work/seohwa` | 감지값 |
| 2 | `FLEET` | 지금 만드는 관제 저장소입니다. 카드·스크립트·로그가 여기 쌓입니다 | `pwd` · `stat -f -c %T .` (ext4 인지) | `/home/com/work/seohwa-agents` | 현재 디렉터리 |
| 3 | `DEV_BRANCH` | 사람이 평소 작업하는 브랜치입니다. 에이전트 결과는 여기로 바로 들어가지 않고 `integration` 을 거칩니다 | `git branch --show-current` | `dev-ryu` | 감지값 |
| 4 | `BASELINE_COMMIT` | 이 커밋까지는 에이전트가 절대 건드리지 않는 기준점입니다. 문제가 생기면 여기로 되돌립니다 | `git log --oneline -5` | `54467c5` | 현재 HEAD |
| 5 | `PROTECTED_BRANCHES` | (위 예시 화면 참조) | `git branch -a` | `dev-ryu,main` | 개발 브랜치 + main·master |
| 6 | `HOTSPOTS` | 여러 에이전트가 동시에 고치면 충돌이 잘 나는 파일입니다. 여기 적으면 한 번에 한 카드만 고치게 순서를 정합니다 | 최근 100커밋에서 가장 자주 바뀐 파일 5개 (`git log --name-only -100`) | `menus=frontend/app/lib/menus.ts` | 비움(나중에 추가 가능) |
| 7 | `DIRTY_EXCEPT` | 커밋 안 된 채 둬도 되는 폴더입니다. 여기 없는 미커밋 파일이 있으면 세팅이 멈춥니다 | `git status --porcelain` 의 미추적 폴더 | `experiments/` | 감지된 후보 |
| 8 | `MODEL_ORCH` · `MODEL_HARD` | 판단·어려운 문제를 맡길 모델입니다 | `claude --model claude-fable-5-1 -p hi` 결과(성공/실패) | `claude-fable-5-1` | 기동에 성공한 가장 높은 모델 |
| 9 | `COMMIT_AUTHOR_NAME` · `COMMIT_AUTHOR_EMAIL` | 에이전트가 만드는 모든 커밋에 찍힐 계정입니다. **이메일은 GitLab·GitHub 에 등록된 것과 같아야** 원격 화면에서 같은 사람으로 보입니다 | `git config user.name` · `git config user.email` · 최근 커밋 작성자 3명 (`git log -20 --format='%an <%ae>' \| sort -u`) | `류현` · `ryu@example.com` | 감지값. **비어 있으면 넘어갈 수 없음** |
| 10 | `COMMIT_COAUTHOR` | 커밋 메시지 끝에 "공동 작성자: Claude" 같은 줄을 붙일지입니다. 0 이면 붙이지 않고, 붙은 커밋은 통합 전에 거부합니다 | — | `0` | 0 |
| 11 | 사무 레인 사용 (y/N) → `OFFICE_*` · `VAULT` · `OUTBOX` | 보고서·공문 같은 문서 작업도 맡길지입니다. y 면 문서 저장소 경로, 읽기 전용 메모 폴더, 완성본 사본을 둘 윈도우 폴더를 이어서 묻습니다 | `ls /mnt/c/Users/*/Obsidian` · `ls /mnt/c/work` | `OUTBOX=/mnt/c/work/산출물` | N |
| 12 | 장기기억 DB 사용 (y/N) → `MEMORY_*` | 에이전트끼리 "지난번에 뭘 정했는지" 를 공유하는 DB 를 둘지입니다. 메모리를 0.3~1.5GB 더 씁니다. y 여도 지금은 설정만 저장하고 DB 는 2-7 에서 띄웁니다 | `free -h` 의 available · 5632 포트 사용 여부 (`ss -ltn`) | `MEMORY_EMBED=none` | N |

### 채우는 순서

**최초 1회만 묻습니다.** `fleet-init.sh` 를 실행하면 0단계에서 `.fleet/config.env` 를 보고, **파일이 없거나 크기가 0이면** 질문을 시작해 값을 채운 뒤 다음 단계로 넘어가고, **크기가 0이 아니면 질문 없이 통과**합니다. 사람이 파일을 직접 채우는 일은 없습니다. 다시 묻게 하려면 `fleet-init.sh --reconfigure` 로 돌리거나 파일을 비웁니다.

1. `scripts/fleet-init.sh` 실행. 0단계가 감지 가능한 값을 먼저 채우고(`HOST_IP` `DEV_BRANCH` `BASELINE_COMMIT` 후보 `FE_DIR`/`BE_DIR` `USE_ALEMBIC` `FE_I18N_DIRS` `BE_COMPOSE_SERVICE` `ENV_TEMPLATE` `REMOTE_NAME` `DIRTY_EXCEPT` 후보), 판단이 필요한 값만 묻습니다.
2. **경로가 `/mnt/` 로 시작하면 여기서 경고합니다** — "작업 경로가 9p 마운트입니다. ext4(`/home/...`)로 옮기면 실측 기준 fe verify 가 4분 16초 → 20.6초가 됩니다. 계속할까요 (y/N)".
3. 질문이 끝나면 채운 값·비운 값을 표로 보여 주고 확인을 받은 뒤 `config.env` 에 씁니다.
4. `DEV_BRANCH`·`BASELINE_COMMIT`·`PROTECTED_BRANCHES` 가 **"어디까지는 건드리지 않는가" 를 정하는 값**이므로 이 셋만은 눈으로 확인합니다.
5. 자기 스택에 없는 것(`BE_DIR`, `USE_ALEMBIC`, `FE_I18N_DIRS`, `HOTSPOTS`)은 감지 단계에서 빈 값이 되고, 빈 만큼 verify·락·티어 판정이 단순해집니다.
6. (4)·(5) 는 기본값으로 시작하고 1~2일 운영 뒤 `config.env` 를 직접 고칩니다.
7. (6) 은 사무 저장소 사용 여부를 y 로 답했을 때만 이어서 묻습니다.
8. **(v4) `COMMIT_AUTHOR_NAME`·`COMMIT_AUTHOR_EMAIL` 은 감지값(`git config user.name/email`)을 보여 주고 반드시 확인받습니다.** 비어 있으면 진행하지 않습니다.
9. (8) 은 장기기억 DB 사용 여부를 y 로 답했을 때만 묻고, 처음에는 `MEMORY_ENABLE=0` 으로 기록한 뒤 2-7 이 끝나면 1 로 바꿉니다.

* * *

## 1. 전체 그림

### 1.1 세 층 모델 배분

| 층 | 모델 | 하는 일 | 절대 하지 않는 일 |
| --- | --- | --- | --- |
| 판단 | **Fable** | 카드 작성·배정·의존성·ADR 승인, 어려운 버그 원인 분석, 문서 구성 검토 | 직접 구현, grep, 로그 전체 읽기 |
| 구현 | **Opus** (문서·디자인은 Opus 1M) | 명세가 정해진 코드 작성·수정·커밋, 문서 작성·수정 | 파일 탐색, 검증 로그 직접 읽기 |
| 잡일 | **Haiku** | 파일·심볼 찾기, grep 요약, 검증 실행 후 실패 항목만 추출, diff·로그 요약, 자료 발췌, 형식 점검 | 코드·문서 수정(Write·Edit 없음) |

### 1.2 역할 (세션) 구성

| 역할 | 개수 | 모델 | 실행 방식 |
| --- | --- | --- | --- |
| ORCHESTRATOR(판단) | 1 | Fable | 루트 세션 또는 `fleet-orch.sh` 헤드리스 |
| DEV-FE(프론트 개발) | 3 | Opus | `wt/<ID>/` worktree 세션 |
| DEV-BE(백엔드 개발) | 3 | Opus | `wt/<ID>/` worktree 세션 (worktree 당 컨테이너 2개) |
| OFFICE(문서작업) | 1~3 | Opus 1M | `wt/OF-*` `wt/GO-*` worktree 세션 |
| DESIGNER(와이어프레임) | 1 | Opus 1M | `.fleet/design/<ID>/` 산출 |
| REVIEWER / QA / INTEGRATOR | 카드당 | Opus / Sonnet / Sonnet | 헤드리스 또는 서브에이전트 |

### 1.3 서브에이전트 11개 (worktree 마다 `.claude/agents/` 에 복사됨)

| 분야 | 이름 | 모델 | 역할 |
| --- | --- | --- | --- |
| 프론트 | `fe-explore` `fe-verify` `fe-summarize` | Haiku | 탐색 / 검증 실행·실패 요약 / diff·로그 요약 |
| 프론트 | `fe-hard` | Fable | 어려운 작업 전담(조건부) |
| 백엔드 | `be-explore` `be-verify` `be-summarize` | Haiku | 위와 동일 |
| 백엔드 | `be-hard` | Fable | 어려운 작업 전담(조건부) |
| 문서 | `doc-find` `doc-check` | Haiku | 자료 검색·발췌 / 형식·필수절·기밀 점검 |
| 문서 | `doc-judge` | Fable | 구성·증빙·심사 기준 검토, **지시만** (Write 없음) |

### 1.4 흐름

```
fe/be:   task-new → wt-new → (memory_search) → [explore(Haiku) → DEV(Opus) → verify(Haiku)] ×n → (2회 실패 시 *-hard(Fable)) → summarize(Haiku) → (memory_write) → review-req(계정 검사) → REVIEWER → integrate(계정 검사) → push(사람)
office:  task-new → wt-new → [doc-find(Haiku) → OFFICE(Opus 1M) → doc-check(Haiku) → doc-judge(Fable)] ×n → verify → review-req → publish → OUTBOX 복사
```

제약: Claude Code 서브에이전트는 다른 서브에이전트를 부를 수 없습니다. 그래서 DEV 세션(최상위)이 자기 worktree 의 서브에이전트를 부르고, 헤드리스 루프는 프롬프트에 "탐색은 `*-explore`, 검증은 `*-verify` 에 위임" 을 박아 넣습니다.

* * *

## 2. 구축 순서 (Claude Code 에 지시문을 그대로 붙여 넣는다)

전체 6단계. 각 단계마다 **Claude 에게 줄 지시문**이 있습니다. `<프로젝트>` `<사용자>` 는 자기 것으로 바꿉니다. 판단이 필요한 단계이므로 **구축 세션은 Fable 또는 Opus 로** 엽니다(`claude --model claude-fable-5-1`).

### 2단계 준비: 관제 저장소 뼈대

```bash
mkdir -p ~/work/<프로젝트>-agents/{roles/agents/{fe,be,office},scripts,office-templates,office-tools,docs,.fleet/{tasks,status,reviews,contracts,daily,locks,logs,backup,inputs,adr,design,report,inbox/done,run},wt}
cd ~/work/<프로젝트>-agents && git init
```

**코드 저장소가 아직 `/mnt/c` 에 있다면 먼저 옮깁니다.** 부록 A 와 같은 절차이며, 저장소 하나만 옮기는 경우는 `rsync -a /mnt/c/work/<프로젝트>/ ~/work/<프로젝트>/` 뒤에 `git -C ~/work/<프로젝트> status` 로 확인하면 끝입니다(worktree 가 아직 없을 때).

### 2-0. 최초 세팅 — 건드리면 안 되는 것과 안전장치 (가장 먼저, 한 번만)

에이전트를 돌리기 전에 **사람이 직접 정하고 잠가 두는 것**들입니다. 여기서 정한 것은 이후 스크립트(`guard-path.sh` · `settings.local.json` deny · `integrate.sh`)가 강제하므로, 빠뜨리면 에이전트가 메인 브랜치를 건드리거나 되돌릴 수 없는 상태가 됩니다.

#### (1) 보호 대상 — 절대 직접 수정·커밋하지 않는 것

| 대상 | 규칙 | 강제하는 곳 |
| --- | --- | --- |
| 메인 저장소 워킹트리 | **읽기 전용 기준점.** 모든 작업은 `wt/<ID>/` worktree 에서만 | `guard-path.sh`(쓰기 차단) · 역할 파일 |
| 개발 브랜치(`PROTECTED_BRANCHES`) | 직접 커밋 금지. 통합은 `integration` 으로만 | `integrate.sh` 가 `integration` 에만 `--no-ff` 머지 |
| 사무 저장소 `main` | 직접 커밋 금지. `office-publish.sh` 만 예외 | `guard-path.sh` |
| 원격 `origin` | **push 는 사람이 직접.** 에이전트 `git push` 금지 (`LOOP_AUTO_PUSH=0` 으로 시작) | `settings.local.json` deny `Bash(git push*)` |
| Obsidian 볼트 | 읽기만 | `guard-path.sh` |
| 사용자 원본 자료(바탕화면 등) | 옮기지도 고치지도 않음. `입력/<ID>/` 에 복사본 | 역할 파일 · verify |
| `package.json` `package-lock.json` `requirements.txt` `docker-compose.yml` `.gitignore` `.gitattributes` | 에이전트 수정 금지. 의존성 추가는 오케스트레이터가 메인 작업본에서만 | deny `npm install*` `pip install*` · 리뷰 게이트 |
| `.env` `.env.local` `SECRET_KEYS` | 읽기·출력 금지 | deny `Read(**/.env)` |
| `backend/data/` `terraform/` `migrations/`(스키마 변경 카드 외) | 수정 금지 | 범위 검사 · `alembic` 락 |
| git 명령 `checkout` `switch` `rebase` `reset --hard` `stash` `worktree` `filter-branch` `--force` | worktree 안에서 금지 | deny 목록 |
| **`OUTBOX` 의 파일** | **사본입니다.** 여기서 고쳐도 저장소에 반영되지 않음 | 역할 파일 · 발행 스크립트 인쇄 문구 |

`integration` 브랜치가 **유일한 통합 지점**입니다. 사람은 `integration` 을 확인한 뒤 개발 브랜치로 옮기거나 push 합니다.

#### (2) 되돌릴 수 있게 — 기준선 태그와 번들

시작 시점의 메인 저장소에 `fleet-baseline-<YYYYMMDD-HHMM>` 태그를 찍고 `git bundle create … --all` 로 `.fleet/backup/` 에 전체 백업을 남깁니다. 무엇이 꼬여도 `git reset --hard fleet-baseline-…`(사람이, 메인 작업본에서) 또는 번들 복원으로 돌아갈 수 있습니다. 워킹트리가 깨끗한 상태에서만 찍습니다.

#### (3) 메인 저장소에 남기지 않는 파일 — `.git/info/exclude`

`.gitignore` 는 건드리지 않고 `.git/info/exclude` 에 다음을 넣습니다: `/CLAUDE.md` `/CLAUDE.local.md` `.claude/` `.fleet-port` `<BE_DIR>/docker-compose.override.yml` `/build/`. worktree 마다 생기는 역할 파일·설정·포트 파일이 커밋에 섞이는 것을 막습니다.

#### (4) 환경 파일

`<BE_DIR>/.env` 가 없으면 `ENV_TEMPLATE` 에서 복사하고 `SECRET_KEYS` 가 비어 있으면 생성해 채웁니다(값은 출력하지 않습니다). worktree 는 이 파일을 복사해 `COMPOSE_PROJECT_NAME` 만 바꿔 씁니다. 프론트 `.env.local` 은 `NEXTAUTH_URL` 만 포트에 맞게 바꿔 복사합니다.

#### (5) ext4 기준의 환경 주의사항 (v2 의 drvfs 절을 대체합니다)

- **`safe.directory` 등록·해제가 필요 없습니다.** 파일 소유자가 자기 계정이라 `dubious ownership` 이 나지 않습니다.
- **venv 를 링크로 우회할 필요가 없습니다.** `$FLEET/.venv` 에 직접 만듭니다(`VENV_REAL` 은 호환용으로 남겨 둡니다).
- **사무 저장소를 `git init` 후 `.git` 복사로 만들 필요가 없습니다.** 평범하게 `git init` 하고 `git config` 로 설정을 씁니다. `filemode=false` 도 필요 없습니다.
- **`FE_BUILD_FALLBACK` 은 비웁니다.** Turbopack 이 정상 동작하므로 webpack 재빌드 경로를 두면 실패를 가리게 됩니다.
- **윈도우 프로그램(Chrome·한컴)이 여는 경로만 `/mnt/c`** 입니다. 변환 스크립트는 `wslpath -w` 로 UNC 를 만들어 넘기고, **경로가 `/mnt/c` 인지 검사하지 않습니다.**
- **Docker Desktop 을 쓴다면** `/usr/bin/docker` 가 `/mnt/wsl/docker-desktop/...` 심볼릭 링크라 Desktop 이 꺼지면 함께 사라집니다. be verify 가 1단계에서 멈추면 이것부터 봅니다.
- **`node_modules` 는 worktree 마다 실물로 둡니다.** Next 16 Turbopack 이 프로젝트 밖 심볼릭 링크를 거부합니다. ext4 에서는 `npm ci` 가 충분히 빨라(`/mnt/c` 에서 10분 이상 걸리던 be worktree 생성이 2.4초) 링크 우회를 유지할 이유가 없습니다. 윈도우에서 설치한 `node_modules` 를 복사해 왔다면 리눅스 네이티브 패키지(`lightningcss` `@tailwindcss/oxide` `@next/swc`)를 `npm pack` 으로 보충하는 스크립트(`fix-natives.sh`)를 둡니다 — `package.json` `package-lock.json` 은 건드리지 않습니다.

#### (6) 세션 경계 — Claude Code 설정

worktree 마다 `.claude/settings.local.json` 에 deny 목록(위 표의 git 명령 · `npm install*` · `pip install*` · `Read(**/.env)`)과 `PreToolUse` 훅(`Edit|Write|MultiEdit` → `guard-path.sh`)을 넣습니다. 훅은 권한 모드와 무관하게 항상 실행되므로 헤드리스(`--permission-mode bypassPermissions`)에서도 경계가 지켜집니다. `guard-path.sh` 는 편집 대상이 카드 `scope` 밖 · 메인 저장소 · 볼트 · 다른 worktree · 사무 저장소 main 이면 exit 2 로 차단하고, `.fleet/debt.md` 와 `.fleet/adr/` 만 어느 세션이든 허용합니다.

**훅 경로는 절대경로로 들어갑니다.** 나중에 fleet 을 옮기면 이 경로가 조용히 깨져 가드가 무력화되므로, 이동 시 반드시 함께 치환합니다(부록 A 의 함정 3).

#### (7) 사무 저장소 2개와 산출물함

`~/work/<프로젝트>-office`(프로젝트 사무, `OF-`) 와 `~/work/office`(일반 사무, `GO-`) 를 만듭니다. 구조는 `입력/<ID>/` · `작업/<종류>/<ID>/` · `산출물/<ID>/`. 각 저장소 루트에 문서 규칙 `CLAUDE.md` 를 두고 `.git/info/exclude` 에 `/CLAUDE.local.md` `.claude/` `/build/` 를 넣습니다.

**산출물함**은 `OUTBOX/<프로젝트 구분>/<ID>/` 입니다(예: `C:\work\산출물\서화\OF-KISA-001\`). `office-publish.sh` 가 발행 마지막에 `cp` 하며, **복사 실패가 발행을 되돌리지 않습니다**(경고만). 사용자가 한컴·엑셀로 여는 곳이 여기 하나뿐이도록 두는 것이 요점입니다.

#### (8) 커밋 계정 통일 (v4)

결론: **커밋 계정은 `.fleet/config.env` 의 `COMMIT_AUTHOR_NAME` · `COMMIT_AUTHOR_EMAIL` 한 곳에서만 정하고, 네 겹으로 강제합니다.** 한 겹만 두면 세션·도구·스크립트 중 어느 하나가 다른 계정을 쓰는 순간 이력이 흩어집니다.

| 겹 | 무엇을 | 어디서 | 왜 |
| --- | --- | --- | --- |
| ① 저장소 설정 | `git config --local user.name/user.email` 을 설정값으로 | `fleet-init.sh` 5b 단계 — 메인 저장소 · 관제 저장소 · 사무 저장소 2개 | worktree 는 부모 저장소의 로컬 설정을 공유하므로 새 worktree 도 자동 적용 |
| ② 환경 변수 | `GIT_AUTHOR_NAME` `GIT_AUTHOR_EMAIL` `GIT_COMMITTER_NAME` `GIT_COMMITTER_EMAIL` export | `_common.sh`(모든 스크립트) · `agent-run.sh`(헤드리스) · worktree `settings.local.json` 의 `env` | 환경 변수가 `git config` 보다 우선하므로, 누가 전역 설정을 바꿔도 에이전트 커밋은 설정값으로 남음 |
| ③ 공동 작성자 줄 차단 | `COMMIT_COAUTHOR=0` 이면 Claude Code 가 커밋 메시지에 붙이는 공동 작성자 줄을 끔 | worktree `settings.local.json` — `includeCoAuthoredBy: false` `[확인 필요: 설치된 Claude Code 버전의 설정 키 이름 — 버전에 따라 attribution 계열 키로 바뀌었을 수 있음. claude 안에서 /config 또는 공식 설정 문서로 확인]` · 역할 파일 금지 문구 | 도구 기본값이 계정 통일을 깨는 가장 흔한 원인 |
| ④ 사후 검사 | 카드 브랜치의 모든 커밋이 설정 계정인지, 공동 작성자 줄이 없는지 검사 | `check-identity.sh` 를 `review-req.sh` · `integrate.sh` · `office-publish.sh` 가 호출 | 앞 세 겹을 우회한 커밋을 통합 전에 잡음 |

검사 명령의 핵심은 다음 두 줄입니다.

```bash
git log --format='%an|%ae|%cn|%ce' "$BASE_BRANCH..$BRANCH" | grep -v -x "$COMMIT_AUTHOR_NAME|$COMMIT_AUTHOR_EMAIL|$COMMIT_AUTHOR_NAME|$COMMIT_AUTHOR_EMAIL"   # 출력이 있으면 위반
git log --format=%B "$BASE_BRANCH..$BRANCH" | grep -i '^co-authored-by:'                                                                                     # COMMIT_COAUTHOR=0 인데 출력이 있으면 위반
```

**위반이 나왔을 때**: 마지막 커밋 하나면 담당 세션이 `git commit --amend --reset-author --no-edit` (환경 변수가 설정값이므로 그 계정으로 다시 찍힘)로 고칩니다. 여러 커밋이면 **사람이 판단**합니다 — `rebase` 는 worktree 안에서 금지 명령이므로 에이전트가 이력을 다시 쓰지 않습니다. `integrate.sh` 가 만드는 `--no-ff` 머지 커밋도 ② 의 환경 변수로 같은 계정이 됩니다.

**어떤 계정을 쓸지**는 팀 규칙에 따릅니다. 사람 계정 하나로 통일하면 원격 화면이 단순하고, 전용 봇 계정을 두면 "에이전트가 만든 커밋" 이 구분됩니다. 어느 쪽이든 **이메일은 원격에 등록된 것**이어야 합니다.

**Claude 에게 줄 지시문 ⓪ (fleet-init.sh)**

```
scripts/fleet-init.sh 를 만들고 실행하십시오. 사용: fleet-init.sh [--reconfigure] [--yes]. 멱등이어야 하며 단계별로 "── N. 제목" 을 출력합니다. 모든 값은 .fleet/config.env(0-1 설정 시트)에서 읽고, 빈 변수는 해당 단계를 "건너뜀" 으로 출력하고 넘어갑니다.
0. 최초 설정(최초 1회만): .fleet/config.env 가 없거나 0바이트이면(또는 --reconfigure) 질문 절차를 실행하고, 아니면 "설정 있음 — 건너뜀" 후 0b 로 갑니다.
   0-1) 감지: REPO 는 인자 또는 현재 디렉터리 상위의 .git 이 있는 곳. **FLEET 은 현재 디렉터리(pwd)**. DEV_BRANCH=git branch --show-current, BASELINE_COMMIT 후보=git rev-parse --short HEAD, REMOTE_NAME=git remote 첫 줄, HOST_IP=hostname -I 첫 값, FE_DIR/BE_DIR/BE_COMPOSE_SERVICE/USE_ALEMBIC/FE_I18N_DIRS/ENV_TEMPLATE 는 파일 존재로 감지, DIRTY_EXCEPT 후보=git status --porcelain 의 미추적 디렉터리.
   0-2) **경로 검사**: REPO·FLEET·OFFICE_* 중 /mnt/ 로 시작하는 것이 있으면 경고합니다 — "작업 경로가 9p 마운트입니다. ext4(/home/...)에서 fe verify 가 4분 16초 → 20.6초로 측정됐습니다(같은 카드·같은 커밋). 계속하시겠습니까 (y/N)". --yes 면 경고만 출력하고 진행합니다.
   0-3) 질문은 0-1 절의 「질문 화면 규격」 6줄 형식(진행 표시 · 무엇 · 왜 · 확인(실제 실행 결과) · 예시 · 입력 [Enter = 기본값])과 「질문 목록과 안내 문구」 표의 문구를 그대로 씁니다. `?` 입력 시 도움말, 잘못된 값은 이유와 함께 재질문(3회까지), 마지막에 4열 요약표와 번호로 고치기를 지원합니다. 질문 순서(--yes 면 감지값 통과): REPO·FLEET 확인 / BASELINE_COMMIT 확정 / PROTECTED_BRANCHES / HOTSPOTS / 사무 저장소 사용 여부(y 면 OFFICE_* 와 VAULT 와 OUTBOX 를 이어서 질문) / DIRTY_EXCEPT / MODEL_ORCH·MODEL_HARD / **COMMIT_AUTHOR_NAME·COMMIT_AUTHOR_EMAIL(감지값 = git config user.name/email 을 보여 주고 확인. --yes 여도 비어 있으면 멈춤)** · COMMIT_COAUTHOR(기본 0) / 장기기억 DB 사용 여부(y 면 MEMORY_* 를 이어서 질문, MEMORY_ENABLE 은 0 으로 기록).
   0-4) 채운 값·비운 값을 표로 출력하고 확인 후 config.env 에 씁니다. SECRET_KEYS 값과 MEMORY_DB_URL 의 비밀번호는 절대 출력하지 않습니다.
0b. config.env 검사: 필수 변수(REPO FLEET DEV_BRANCH HOST_IP COMMIT_AUTHOR_NAME COMMIT_AUTHOR_EMAIL, be 레인을 쓰면 BE_COMPOSE_SERVICE, MEMORY_EMBED=local 이면 MEMORY_EMBED_MODEL)가 비어 있거나 경로가 없으면 목록을 출력하고 종료.
1. 도구 점검: git node npm python3(venv) docker pandoc claude. BE_DIR 이 비면 docker 는 검사하지 않음.
1b. **파일시스템 점검**: stat -f -c %T "$FLEET" 로 ext4 인지 확인하고, 9p 이면 경고를 남깁니다. ext4 이면 "safe.directory·venv 링크·filemode 우회 불필요" 를 출력하고 해당 단계들을 건너뜁니다.
2. 메인 저장소 워킹트리가 깨끗한지 검사(DIRTY_EXCEPT 무시). 더러우면 목록 출력 후 종료.
3. 기준선: BASELINE_COMMIT(없으면 DEV_BRANCH HEAD)에 fleet-baseline-<YYYYMMDD-HHMM> 태그를 찍고 git bundle create .fleet/backup/<프로젝트>-<시각>.bundle --all.
4. BASE_BRANCH 가 없으면 기준선 커밋에서 생성하고 wt/integration worktree 를 만듭니다. PROTECTED_BRANCHES 를 .fleet/protected.txt 에 기록합니다.
4b. wt/integration 의 로컬 환경: FE_DIR 이 있으면 node_modules(npm ci)와 .env.local(NEXTAUTH_URL=INT_FE_PORT), BE_DIR 이 있으면 .env(COMPOSE_PROJECT_NAME=<프로젝트>-integration)와 docker-compose.override.yml(INT_API_PORT·INT_PG_PORT).
5. .git/info/exclude 에 6항목 추가(.gitignore 는 건드리지 않음).
5b. **커밋 계정**: REPO·FLEET 과 설정된 OFFICE_* 저장소 각각에 git config --local user.name "$COMMIT_AUTHOR_NAME" · user.email "$COMMIT_AUTHOR_EMAIL" 을 씁니다. 기존 로컬 값이 다르면 전·후 값을 출력합니다(전역 ~/.gitconfig 는 건드리지 않음). 끝나면 각 저장소에서 git config user.email 을 다시 읽어 일치하는지 보고합니다.
6. ENV_TEMPLATE 이 있고 <BE_DIR>/.env 가 없으면 복사, SECRET_KEYS 가 비어 있으면 키 생성해 채움(값 출력 금지).
7. 사무 도구(OFFICE_SEOHWA 가 있을 때만): $FLEET/.venv 에 venv 를 만들고 python-docx openpyxl python-pptx fpdf2 설치, pyhwp 는 HWP_INPUT=1 일 때만 시도. office-templates/(OFFICE_KINDS 별, 첫머리 "필수절:" 주석)과 office-tools/(md2docx.sh md2pptx.sh csv2xlsx.py hwp2txt.sh) 생성. **변환 스크립트에 출력 경로가 /mnt/c 인지 검사하는 조건을 넣지 마십시오** — wslpath -w 로 UNC 를 만들어 윈도우 Chrome 에 넘기면 ext4 에서도 동작합니다.
8. 사무 저장소 생성(경로가 설정된 것만): git init · 루트 CLAUDE.md · .git/info/exclude. ext4 이므로 .git 복사나 filemode=false 는 필요 없습니다.
8b. OUTBOX 가 설정돼 있으면 OUTBOX/<프로젝트 구분> 폴더를 만들고 쓰기 가능 여부를 확인합니다.
9. Obsidian 볼트는 건드리지 않음(VAULT_GIT=0).
10. .fleet/ 하위 디렉터리와 board.md, debt.md, adr/000-템플릿.md 생성. HOTSPOTS 가 있으면 .fleet/locks/ 를 만들고 이름 목록을 기록.
11. 장기기억 DB 사용을 y 로 답했다면 .fleet/memory/ 와 .fleet/mcp/ 디렉터리만 만들고 "2-7 지시문 ⑦ 에서 구성" 이라고 출력합니다(여기서 DB 를 띄우지 않음).
끝나면 각 단계 결과(실행/건너뜀/실패)를 두괄식으로 보고하십시오.
```

### 2-1. 공통 규칙과 설정 파일

**Claude 에게 줄 지시문 ①**

```
~/work/<프로젝트>-agents 에 멀티에이전트 관제 저장소를 만듭니다. 코드 저장소는 ~/work/<프로젝트> (frontend: Next.js, backend: FastAPI+Postgres docker compose) 이고 읽기 전용 기준점입니다.

1) 루트 CLAUDE.md 를 만드십시오. 내용: 존댓말·두괄식 원칙 / 작업은 자기 worktree 안에서만, 메인 저장소·Obsidian 볼트·다른 worktree 는 읽기 전용 / 산출물함(OUTBOX)은 사본이며 정본은 저장소 안 산출물/<ID>/ / 금지 사항(git push·checkout·rebase·reset --hard·stash·worktree 명령, npm install·pip install, .env 출력, 린트·타입 오류를 규칙 비활성화로 무마) / 착수 전·완료 후 .fleet/board.md 확인 / 완료 절차: verify.sh 통과 → review-req.sh → 리뷰어 '판정: approved' → 통합 / 설계 규율(새 기능은 분기 추가가 아니라 모듈 추가, 3회 반복 시 추출, 60줄 함수 분리, 매직 값 금지, 의존 방향 page→lib/hook→api / openapi→engine→infra) / 모델 배분과 위임 규칙(아래 3절 그대로) / **커밋 규칙(v4): 커밋 계정은 환경 변수로 고정돼 있으니 git config 를 바꾸지 말 것, 커밋 메시지에 Co-Authored-By 등 공동 작성자 줄을 넣지 말 것(COMMIT_COAUTHOR=0), --author 옵션 사용 금지** / **장기기억 규칙(v4, MEMORY_ENABLE=1 일 때): 착수 시 memory_search, 완료 시 memory_write, DB 는 정본이 아니며 파일(카드·ADR·보드)과 다르면 파일을 따른다, 비밀 값 저장 금지**.

2) .fleet/config.env 는 직접 만들지 않습니다. fleet-init.sh 0단계가 만듭니다. 그 스크립트가 쓸 "기본 블록" 은 아래이며, 여기에 0-1 설정 시트의 변수 전부를 (1)~(7) 순서로 주석 구분선과 함께 붙이고 값이 없는 항목은 빈 값으로 둡니다. 이후 만들 모든 스크립트는 경로·브랜치 이름·핫스팟·커밋 접두사·서비스명·검증 순서를 절대 하드코딩하지 말고 이 변수를 읽습니다.
REPO=/home/<사용자>/work/<프로젝트>
FLEET=/home/<사용자>/work/<프로젝트>-agents
OFFICE_SEOHWA=/home/<사용자>/work/<프로젝트>-office
OFFICE_GENERAL=/home/<사용자>/work/office
VAULT=/mnt/c/Users/<이름>/Obsidian/업무
OUTBOX=/mnt/c/work/산출물
DEV_BRANCH=<개발 브랜치>
BASE_BRANCH=integration
BASELINE_COMMIT=
PROTECTED_BRANCHES=<개발 브랜치>,main,master
REMOTE_NAME=origin
FE_PORT_BASE=3101
BE_API_PORT_BASE=8101
BE_PG_PORT_BASE=5433
MAX_WT_FE=3
MAX_WT_BE=3
MAX_WT_OFFICE=3
MAX_CONCURRENT_AGENTS=2
HOST_IP=<WSL에서 보이는 PC IP>
LOOP_TICK=60
LOOP_REPORT_EVERY=3600
LOOP_MAX_TRIES=3
LOOP_AUTO_PUSH=0
LOOP_TIER_FILES=3
LOOP_TIER_LINES=200
AGENT_TIMEOUT=3600
MODEL_ORCH=claude-fable-5-1
MODEL_DEV=opus
MODEL_OFFICE=opus[1m]
MODEL_DESIGN=opus[1m]
MODEL_REVIEW=opus
MODEL_HELPER=haiku
MODEL_HARD=claude-fable-5-1
HARD_AFTER_TRIES=3
# ── (v4) 커밋 계정
COMMIT_AUTHOR_NAME=<커밋 계정 이름>
COMMIT_AUTHOR_EMAIL=<원격에 등록된 이메일>
COMMIT_COAUTHOR=0
COMMIT_IDENTITY_STRICT=1
# ── (v4) 장기기억 DB
MEMORY_ENABLE=0
MEMORY_PG_PORT=5632
MEMORY_DB_URL=
MEMORY_EMBED=none
MEMORY_EMBED_MODEL=
MEMORY_TOPK=8
MEMORY_TTL_DAYS=90
MEMORY_WRITE_FACT_ROLES=orchestrator,integrator
```

모델 별칭: `opus` `sonnet` `haiku` `opus[1m]` 은 Claude Code 별칭이고, Fable 은 별칭이 없어 전체 ID `claude-fable-5-1` 을 씁니다. 자기 계정에서 쓸 수 있는 ID 는 `claude` 실행 후 `/model` 로 확인합니다.

### 2-2. 운영 스크립트

**Claude 에게 줄 지시문 ②**

```
~/work/<프로젝트>-agents/scripts/ 에 bash 스크립트를 만드십시오. 모두 첫 줄에서 _common.sh 를 source 하고 .fleet/config.env 를 읽습니다. 각 스크립트는 사용법 주석 한 줄로 시작합니다. **경로를 하드코딩하지 마십시오** — FLEET 을 상수로 박으면 나중에 저장소를 옮길 때 조용히 깨집니다.

- _common.sh: config.env 로드 후 **GIT_AUTHOR_NAME/GIT_AUTHOR_EMAIL/GIT_COMMITTER_NAME/GIT_COMMITTER_EMAIL 를 COMMIT_AUTHOR_* 로 export(v4)**, die/warn/info, card_get <ID> <키>, lane_of/kind_of/project_of, wt_path, status_path/status_get/status_update, active_ids_for_lane, smallest_free_port, lock_owner, fleet_commit, base_branch_of, repo_of, outbox_dir_of(프로젝트 구분 → OUTBOX 하위 폴더)
- task-new.sh <ID> --lane fe|be|office [--kind] [--source] [--to] [--due] [--effort] [--schema-change] [--depends]: .fleet/tasks/<ID>.md 카드
- wt-new.sh <ID>: 카드 확인 → 레인 상한 확인 → git worktree add → 포트 배정(.fleet-port) → roles/<역할>.md 를 {{ID}} {{WT}} {{PORT}} {{BRANCH}} {{SCOPE}} 치환해 worktree 의 CLAUDE.md(office 는 CLAUDE.local.md)로 렌더 → .claude/settings.local.json 생성 → roles/agents/<lane>/*.md 를 <worktree>/.claude/agents/ 로 복사 → 상태 JSON 생성 → 세션 시작 명령과 첫 지시문 인쇄. fe 는 node_modules 를 npm ci 로 실물 설치합니다(ext4 에서는 빠릅니다).
  settings.local.json 은 {"model": "<레인별 MODEL_*>", "env": {"GIT_AUTHOR_NAME": …, "GIT_AUTHOR_EMAIL": …, "GIT_COMMITTER_NAME": …, "GIT_COMMITTER_EMAIL": …}, (COMMIT_COAUTHOR=0 이면) "includeCoAuthoredBy": false [확인 필요: 설치 버전의 키 이름], "permissions": {"deny": [... , "Bash(git config*)", "Bash(git commit*--author*)"]}, "hooks": {"PreToolUse": [{"matcher": "Edit|Write|MultiEdit", "hooks": [{"type":"command","command":"<FLEET>/scripts/guard-path.sh"}]}]}} 입니다.
- wt-rm.sh <ID>: worktree 제거(브랜치 유지), be 는 compose down, 상태를 archived 로. **archived 로 바꾸지 않으면 레인 상한을 계속 차지합니다.**
  레인 상한 계산(`active_ids_for_lane`)은 `state != archived` 를 전부 활성으로 세므로, 통합이 끝나 `merged` 로 멈춘 카드도 자리를 차지합니다. 통합 직후 archived 처리를 루프에 넣으십시오.
- verify.sh 는 상태를 바꾸는 스크립트입니다. **이미 통합된 카드에 검증 목적으로 다시 돌리면 상태가 in-progress 로 되돌아가 보드가 틀어집니다.** 성능 측정이나 환경 점검으로 돌릴 때는 verify-fe.sh/verify-be.sh 를 <ID> <WT> <LOGDIR> 인수로 직접 부르십시오(상태 파일을 건드리지 않습니다).
- guard-path.sh: PreToolUse 훅. 편집 대상 경로가 카드 scope 밖·메인 저장소·볼트·다른 worktree·사무 저장소 main 이면 exit 2
- verify.sh <ID>: 레인별 verify-fe.sh(tsc·lint·i18n·build) / verify-be.sh(import·pytest·compose 재시작·openapi 200·스키마 변경 시 빈 볼륨 upgrade) / verify-office.sh(필수 절·두괄식·존댓말·기밀·변환 빌드) + 범위 검사. 결과를 상태 JSON 에 기록. **빌드 폴백(webpack 재시도)을 넣지 마십시오** — ext4 에서 Turbopack 이 실패하면 그것은 진짜 실패입니다.
- i18n-check.mjs: messages/ko.json 과 en.json 의 키 경로 집합 비교, 차이가 있으면 exit 1
- fix-natives.sh <frontend 디렉터리>: 윈도우에서 설치된 node_modules 를 가져온 경우 -linux-x64-gnu 패키지만 npm pack 으로 보충(package.json·lock 은 건드리지 않음)
- **check-identity.sh <ID> (v4)**: 카드 브랜치의 BASE_BRANCH..BRANCH 커밋 전부가 COMMIT_AUTHOR_NAME/EMAIL(작성자·커미터 모두)인지, COMMIT_COAUTHOR=0 이면 공동 작성자 줄이 없는지 검사. 위반 커밋을 해시·계정으로 출력하고 COMMIT_IDENTITY_STRICT=1 이면 exit 1. 마지막 커밋 하나만 위반이면 고치는 명령(git commit --amend --reset-author --no-edit)을 안내.
- review-req.sh <ID>: verify 통과 확인 → **check-identity.sh 통과 확인(v4)** → .fleet/reviews/<ID>.md 와 <ID>.diff 생성, 상태 review-requested
- integrate.sh <ID>: verify 통과·판정 approved·락 미보유·**check-identity 통과(v4)** 확인 → integration 에 --no-ff 머지 → 재검증 → 실패 시 ORIG_HEAD 롤백
- office-publish.sh <ID>: 사무 worktree → 사무 저장소 main 머지 + md2docx/pptx/xlsx 변환 → 산출물/<ID>/ → **OUTBOX/<프로젝트 구분>/<ID>/ 로 cp(실패해도 발행은 되돌리지 않고 경고만)**
- local-stack.sh up|down|rebuild|status|logs: 통합본(wt/integration)으로 로컬 실행 스택 기동
- lock.sh acquire|release|status <핫스팟> <ID>
- board.sh: .fleet/board.md 갱신(진행 중 카드·상태·락·포트)
- agent-run.sh <ID> dev|reviewer: 헤드리스 1회 실행(아래 2-3 규격)
- fleet-orch.sh <명령파일>: claude -p "$P" --model "$MODEL_ORCH" --permission-mode bypassPermissions
- fleet-loop.sh / fleet-stop.sh / fleet-tell.sh: 자율 운행 루프(상태 기계, 재시도 상한, 티어 A 자동 통합, 인박스 명령)
- office-daily.sh [--vault]: .fleet/daily/<오늘>.md 생성. 볼트는 --vault 를 명시할 때만 건드림
- (v4 · MEMORY_ENABLE 과 무관하게 만들어 둠) memory-up.sh up|down|status|backup · memory-sync.sh [--since <날짜>] · memory-gc.sh — 내용은 2-7 지시문 ⑦

스크립트를 만든 뒤 bash -n 으로 문법 검사를 하고, 인자 없이 실행하면 사용법을 출력하는지 확인하십시오.
```

### 2-3. `agent-run.sh` 의 모델·에스컬레이션 규격 (핵심)

**Claude 에게 줄 지시문 ③**

```
scripts/agent-run.sh 는 다음 규격을 정확히 지키십시오.

1) 모델 선택
   ROLE=reviewer → MODEL=$MODEL_REVIEW
   LANE=fe|be    → MODEL=$MODEL_DEV
   LANE=office   → MODEL=$MODEL_OFFICE
2) *-hard 강제 조건(하나라도 참이면 HARD=1)
   - $FLEET/.fleet/run/<ID>.tries.dev 의 값 ≥ $HARD_AFTER_TRIES (직전 2회 실패)
   - 카드 kind 가 refactor
   - 카드 schema_change 가 true, 또는 .fleet/contracts/<ID>.* 가 존재(티어 B)
3) 프롬프트 구성
   - 공통: "파일·심볼 탐색은 <lane>-explore 에, 검증 실행과 로그 읽기는 <lane>-verify(문서는 doc-check)에, diff·로그 요약은 <lane>-summarize 에 위임하십시오. 직접 grep 하거나 전체 로그를 읽지 마십시오."
   - HARD=1 이면 앞에 추가: "먼저 <lane>-hard 서브에이전트에게 원인 분석과 수정 방향을 요청하고, 그 지시대로 구현하십시오."
   - office 는 공통 대신: "자료 검색·발췌는 doc-find 에, 형식 점검은 doc-check 에 위임하고, 초안 완성 후와 수정 반영 후에는 반드시 doc-judge 에게 검토를 요청해 지시를 반영하십시오."
   - 끝: "사용자에게 질문하지 말고 끝까지 진행하십시오. 완료 보고 마지막 줄에 '서브에이전트 호출: explore n · verify n · hard n' 형식으로 호출 횟수를 적으십시오."
4) 실행: ( cd "$CWD" && timeout "$TIMEOUT" claude -p "$P" --model "$MODEL" --permission-mode bypassPermissions ) >> "$LOG" 2>&1
5) 로그 첫 줄에 model=… hard=… 를, 마지막 줄에 종료 시각과 rc 를 찍어 나중에 배분과 중단을 감사할 수 있게 합니다.
6) **프롬프트에 다음 문장을 반드시 넣으십시오**: "verify·빌드·컨테이너 기동은 포그라운드에서 타임아웃을 주고 실행하십시오. 백그라운드로 돌리고 기다리지 마십시오."
   `claude -p` 는 세션이 끝날 때 백그라운드 작업을 600초까지만 기다리고 종료합니다(`Background tasks still running after 600s; terminating`).
   검증을 백그라운드로 돌리면 결과를 받지 못한 채 세션이 끝나고, 카드는 verify 없는 상태로 남습니다(2026-09-15 실제 발생).
7) **동시 실행 상한을 지킵니다.** 루프나 오케스트레이터가 이 스크립트를 부를 때 살아 있는 agent-run 프로세스 수가 `MAX_CONCURRENT_AGENTS` 이상이면 대기시킵니다.
   상한 없이 6개를 동시에 띄웠다가 메모리 포화로 전부 강제 종료된 사례가 있습니다(0-0 (1)).
8) **(v4) 커밋 계정**: 실행 전에 GIT_AUTHOR_*/GIT_COMMITTER_* 가 COMMIT_AUTHOR_* 로 export 돼 있는지 확인하고, 아니면 실행하지 않습니다. 로그 첫 줄에 commit_as=<이름> <이메일> 을 함께 찍습니다. 프롬프트 끝에 "커밋 계정을 바꾸지 말고, 커밋 메시지에 공동 작성자 줄을 넣지 마십시오." 를 넣습니다(COMMIT_COAUTHOR=0 일 때).
9) **(v4) 장기기억**: MEMORY_ENABLE=1 이고 memory-up.sh status 가 정상일 때만 claude 에 --mcp-config "$FLEET/.fleet/mcp/memory.json" 을 추가하고, 환경 변수 FLEET_ROLE=<dev|reviewer|office…> FLEET_CARD=<ID> 를 넘깁니다. 프롬프트 앞에 "착수 전에 memory_search 로 이 카드 ID·scope·핵심 키워드 관련 기억을 최대 $MEMORY_TOPK 개 조회해 참고하십시오. 기억이 카드·ADR 파일과 다르면 파일을 따르십시오." 를, 끝에 "완료 보고 직전에 memory_write(type=episode) 로 한 일·막힌 곳·해결 방법을 5줄 이내로 남기십시오. 비밀 값은 적지 마십시오." 를 넣습니다. DB 가 내려가 있으면 이 옵션과 문장을 빼고 경고만 로그에 남깁니다(기억이 없어도 작업은 진행).
```

### 2-4. 역할 파일 8종

**Claude 에게 줄 지시문 ④**

```
roles/ 에 역할 파일을 만드십시오. 각 파일은 "# 역할: <이름>" 제목 아래 1.경계 2.착수 전 필수 읽기 3.작업 규율 4.완료 절차 5.하지 말 것 다섯 절로 구성합니다. {{ID}} {{WT}} {{PORT}} {{BRANCH}} {{SCOPE}} {{RF_NOTE}} 자리표시자를 씁니다.

- orchestrator.md: 카드 작성·배정·의존성·계약 조정·ADR 승인만, 직접 구현 금지. 사무 요청은 프로젝트/일반 → kind → 수신자·기한·입력 자료 순으로 카드에 고정. 자율 루프 설명. **레인을 하나만 돌리지 않습니다** — fe 단독 카드와 be 단독 카드를 짝지어 냅니다.
- dev-fe.md / dev-be.md: 경계(worktree·브랜치·포트), 필수 읽기(프로젝트 CLAUDE.md·카드·보드·계약·ADR), 코드 규율, 핫스팟 락, 완료 절차(verify → review-req → 락 해제). 3절 끝에 아래 "위임 규칙" 블록을 그대로 넣습니다.
- office.md: kind 별 템플릿에서 시작, 첫 절은 결론, 근거 없는 값은 [확인 필요], 기밀 값 금지, md 로 쓰고 변환은 스크립트. **산출물함은 사본이며 거기서 고친 것은 반영되지 않는다**는 문장을 1절 경계에 넣습니다. 3절 끝에 "문서 위임 규칙" 블록.
- designer.md: 기존 FE 코드에서 추출한 디자인 토큰만 사용, 저충실도, 라우트당 HTML 1개, 코드 구현 금지.
- reviewer.md / qa.md / integrator.md: 파일 수정 금지(리뷰 문서·debt.md 만), 체크리스트 항목별 판정, 마지막 줄 '판정: approved|changes-requested' / 수용 기준을 실제 실행으로 확인 / integrate.sh·office-publish.sh 만 실행, push 금지.

[위임 규칙 — dev-fe.md · dev-be.md 3절 끝]
- **탐색은 직접 하지 않습니다.** 파일·심볼·사용처 찾기와 grep 은 `<lane>-explore` 서브에이전트에 맡기고 요약만 받습니다.
- **검증 로그를 직접 읽지 않습니다.** tsc·lint·pytest·verify.sh 실행과 실패 항목 추출은 `<lane>-verify` 에 맡깁니다.
- diff·로그 요약, 커밋 메시지 초안, 완료 보고 초안은 `<lane>-summarize` 에 맡깁니다.
- **같은 오류로 두 번 막히거나, 원인을 모르거나, 변경이 5개 파일을 넘을 것 같거나, 구조 변경(ADR)이 필요하면 `<lane>-hard` 에 원인 분석·수정 방향을 요청**하고 그 지시대로 구현합니다.
- 백그라운드 결과를 기다리며 턴을 끝내지 않습니다. 검증은 포그라운드에서 타임아웃을 두고 돌리거나, 10분 이내의 유한 루프로 결과 파일을 직접 확인합니다.
- 완료 보고 마지막 줄에 서브에이전트 호출 횟수를 적습니다.

[문서 위임 규칙 — office.md 3절 끝]
- 자료 검색·발췌는 `doc-find` 에 맡기고 발췌본만 받습니다. 원본 전체를 읽지 않습니다.
- 목차 번호·표기 통일·필수 절 누락·기밀 값 검사는 `doc-check` 에 맡깁니다.
- **초안이 완성되면, 그리고 수정을 반영할 때마다 `doc-judge` 에게 구성·증빙 대응·심사 기준 검토를 요청**하고 지시를 반영합니다. 작성은 이 역할이, 판단은 `doc-judge` 가 합니다.
```

### 2-5. 서브에이전트 11개

**Claude 에게 줄 지시문 ⑤**

```
roles/agents/fe/ be/ office/ 아래에 Claude Code 서브에이전트 정의(md, YAML 프런트매터 name/description/model/tools)를 만드십시오. 본문은 5줄 이내로 짧게, 결론부터 보고하는 형식을 지정합니다. wt-new.sh 가 레인에 맞는 폴더를 <worktree>/.claude/agents/ 로 복사합니다. 아래 정의를 그대로 쓰십시오(be 는 fe 를 복사해 이름·명령만 백엔드로 바꿉니다).
```

```markdown
--- roles/agents/fe/fe-explore.md ---
---
name: fe-explore
description: 프론트 파일·심볼·사용처 찾기와 grep 결과 요약 전담. 구현 전에 "어느 파일의 어느 함수를 고칠지" 목록이 필요할 때 반드시 호출하십시오. 코드를 수정하지 않습니다.
model: haiku
tools: Read, Grep, Glob, Bash
---
결론부터 씁니다. 관련 파일 목록(경로 · 역할 · 고칠 지점 행 번호) → 사용처 → 주의할 의존. 파일 전문은 싣지 않습니다. 관련 파일이 5개를 넘으면 첫 줄에 "관련 파일 N개 — *-hard 검토 권장" 을 씁니다.

--- roles/agents/fe/fe-verify.md ---
---
name: fe-verify
description: tsc·lint·i18n-check·verify.sh 를 실행하고 실패 항목만 파일:행으로 요약합니다. 구현 후·커밋 전에 반드시 호출하십시오. 코드를 고치지 않습니다.
model: haiku
tools: Bash, Read
---
결론 한 줄(통과/실패, 실패 개수) → 실패 항목(파일:행 · 메시지 · 추정 원인 한 줄). 통과한 항목과 전체 로그는 싣지 않습니다. 같은 원인이 반복되면 묶어서 한 항목으로 씁니다.

--- roles/agents/fe/fe-summarize.md ---
---
name: fe-summarize
description: git diff·로그·컨테이너 출력 요약, 커밋 메시지 초안(Feat:/Fix:/Chor:/Refactor:), 카드 완료 보고 초안 작성. 커밋 직전과 review-req 직전에 호출하십시오.
model: haiku
tools: Read, Bash
---
결론부터: 변경 파일 수·추가/삭제 줄 → 파일별 한 줄 요약 → 커밋 메시지 초안 → 카드 수용 기준별 충족 여부. 코드 인용은 5줄 이내.

--- roles/agents/fe/fe-hard.md ---
---
name: fe-hard
description: 원인 불명 버그, 여러 파일(5개 이상)에 걸친 변경, 리팩토링(동작 변경 0건), 구조 판단(ADR) 전담. 같은 오류로 2회 실패했거나 원인을 모를 때, RF 카드일 때, 티어 B 카드일 때 반드시 호출하십시오. 일반 구현에는 호출하지 않습니다.
model: claude-fable-5-1
tools: Read, Grep, Glob, Bash
---
원인과 수정 방향을 두괄식으로 냅니다. 직접 구현하지 않고 "어느 파일을 어떻게, 순서는, 검증 방법은" 까지만 지정합니다. 구조 변경이면 ADR 5줄(맥락·결정·버린 대안·결과·영향 파일) 초안을 함께 냅니다.

--- roles/agents/office/doc-find.md ---
---
name: doc-find
description: 입력 폴더·Obsidian 볼트·증빙 폴더에서 주제와 관련된 파일을 찾아 필요한 부분만 발췌합니다. 초안 작성 전과 절을 채울 때마다 호출하십시오. 원본을 수정하지 않습니다.
model: haiku
tools: Read, Grep, Glob
---
결론부터: 찾은 파일 목록(경로 · 관련 이유) → 파일별 발췌(필요한 문단·표만, 출처 경로:행 표기). 발췌는 전체 2,000자 이내. 없는 사실은 만들지 않고 "[확인 필요: 무엇]" 으로 남깁니다.

--- roles/agents/office/doc-check.md ---
---
name: doc-check
description: 문서의 목차 번호·표기 통일·템플릿 필수 절 누락·[확인 필요] 잔여·기밀 값(키·비밀번호·주민번호·계좌) 을 점검해 목록으로 냅니다. 초안 완성 후와 verify 전에 호출하십시오. 문서를 고치지 않습니다.
model: haiku
tools: Read, Grep
---
결론 한 줄(문제 개수) → 항목별(위치 · 문제 · 고칠 방향 한 줄). 문제 없는 항목은 싣지 않습니다.

--- roles/agents/office/doc-judge.md ---
---
name: doc-judge
description: 보고서·제안서의 구성, 증빙과 항목의 대응, 심사·평가 기준 충족 여부를 검토해 "어느 절을 어떻게 고쳐라" 지시만 냅니다. 초안 완성 직후와 수정 반영 직후 반드시 호출하십시오. 직접 작성·수정하지 않습니다.
model: claude-fable-5-1
tools: Read, Grep, Glob
---
결론부터: 제출 가능 여부(가능/보완 필요) → 보완 지시(절 · 문제 · 고칠 방향 · 근거 자료 위치) 우선순위순 → 증빙 누락 목록. 지시는 항목당 두 줄 이내. 문장을 대신 써 주지 않습니다.
```

### 2-6. 기존 프로젝트에 맞추기와 첫 시험

**Claude 에게 줄 지시문 ⑥**

```
1) 2-0 의 fleet-init.sh 가 아직 안 돌았으면 먼저 실행하고 결과(경로 검사 · 기준선 태그 · integration 브랜치 · exclude · .env · 사무 저장소 · OUTBOX)를 확인하십시오.
2) 시험 카드를 하나 만드십시오: scripts/task-new.sh TASK-SMOKE-001 --lane fe --effort 1 (수용 기준: 특정 컴포넌트에 주석 한 줄 추가, verify 통과).
3) scripts/wt-new.sh TASK-SMOKE-001 을 실행하고 다음을 확인해 보고하십시오:
   - wt/TASK-SMOKE-001/.claude/settings.local.json 의 model 이 "opus" 이고, 훅 command 가 현재 FLEET 경로인가
   - wt/TASK-SMOKE-001/.claude/agents/ 에 fe-explore fe-verify fe-summarize fe-hard 4개가 있는가
   - wt/TASK-SMOKE-001/CLAUDE.md 에 위임 규칙 블록이 있는가
   - worktree 생성에 걸린 시간(ext4 라면 수 초여야 합니다)
4) scripts/agent-run.sh TASK-SMOKE-001 dev 를 실행하고 로그 첫 줄의 model=opus 와, 완료 보고 마지막 줄의 서브에이전트 호출 횟수를 확인하십시오.
5) echo 3 > .fleet/run/TASK-SMOKE-001.tries.dev 를 한 뒤 agent-run.sh 를 다시 돌려 프롬프트에 fe-hard 강제 문구가 들어갔는지 로그로 확인하십시오.
6) fe verify 를 1회 돌려 소요 시간을 기록하십시오(ext4 기준 30초 안쪽이 정상 범위이고, 분 단위면 빌드 폴백이나 9p 경로를 의심합니다).
7) 사무 레인을 쓴다면 시험 발행을 1회 하고, 출력이 "pdf 생성(chrome headless)" 인지 확인하십시오. "fpdf2 폴백" 이면 변환 스크립트에 경로 조건이 남아 있는 것입니다.
8) **메모리 점검**: 시험 카드를 띄운 상태에서 free -h 와 docker ps 를 찍어 보고하십시오. 세션 1개 + 컨테이너가 떠 있을 때 available 이 2GB 미만이면 MAX_WT_* 와 MAX_CONCURRENT_AGENTS 를 낮춰야 합니다(0-0 (1) 산정식).
9) **(v4) 커밋 계정 점검**: 시험 카드 worktree 에서 만든 커밋에 대해 git log -1 --format='%an <%ae> / %cn <%ce>' 가 설정 계정과 같은지, 커밋 메시지에 공동 작성자 줄이 없는지 확인하십시오. 이어서 일부러 GIT_AUTHOR_EMAIL=wrong@example.com 으로 빈 커밋(--allow-empty)을 하나 만든 뒤 check-identity.sh 가 exit 1 로 거부하는지 보고, 그 커밋을 git commit --amend --reset-author --no-edit 로 고친 뒤 통과하는지 확인하십시오.
10) 끝나면 scripts/wt-rm.sh TASK-SMOKE-001 로 정리하고, 상태가 archived 인지 확인하십시오.
```

여기까지 통과하면 구축 완료입니다(장기기억 DB 는 2-7 에서 선택적으로 이어갑니다). `~/.claude/agents/` 에 세션 공용 서브에이전트(`fleet-reviewer` model: opus · `fleet-qa` `fleet-integrator` model: sonnet)를 두면 루트 세션에서도 같은 절감이 됩니다.

### 2-7. 장기기억 DB 연동 (v4 · 선택)

결론: **Postgres + pgvector 컨테이너 하나를 공용 기억 창고로 두고, MCP 서버 하나로 `memory_search` · `memory_write` 두 도구만 열어 모든 세션이 같은 창고를 쓰게 합니다.** 임베딩(벡터 검색)은 나중에 켜도 됩니다 — 처음에는 전문 검색 + 태그만으로 시작하는 편이 준비가 빠르고 메모리도 덜 씁니다.

#### (1) 이론 한 줄

세션의 대화 기록(단기기억)은 끝나면 사라지므로, **작업 결과·결정·사실을 외부 DB 에 남기고 다음 작업을 시작할 때 관련된 것만 꺼내 쓰는 방식**으로 여러 에이전트의 기억을 이어 붙입니다.

#### (2) 무엇을 저장하나 — 기억 종류 4가지

| type | 내용 | 누가 쓰나 | 만료 |
| --- | --- | --- | --- |
| `episode` | 카드별 작업 기록 — 한 일 · 막힌 곳 · 해결 방법 · 커밋 해시 | 모든 세션 | `MEMORY_TTL_DAYS` |
| `fact` | 확인된 사실 — "테스트베드는 스페로우로만" 같은 것 | `MEMORY_WRITE_FACT_ROLES` 만 | 없음(대체될 때까지) |
| `decision` | 결정 — ADR 요약과 파일 위치 | `memory-sync.sh` 가 ADR 에서 자동 적재 | 없음 |
| `procedure` | 잘 된 작업 절차 — "be verify 가 1단계에서 멈추면 Docker Desktop 확인" | 오케스트레이터 · `*-hard` 결과를 오케스트레이터가 승격 | 없음 |

**정본은 파일입니다.** `decision` 은 `.fleet/adr/` 에서, 인계 내용은 `.fleet/daily/*_인계.md` 에서 **파일 → DB 한 방향**으로만 적재합니다. DB 에서 파일을 고치는 경로는 만들지 않습니다.

#### (3) 스키마 — 테이블 3개

결론: **본문(`memories`) · 근거(`memory_sources`) · 벡터(`memory_embeddings`) 세 테이블로 나눕니다.** 임베딩 모델을 바꿔도 본문 테이블 구조를 고칠 필요가 없고, 근거 파일이 바뀌었을 때 영향받는 기억을 바로 찾을 수 있습니다. 처음에는 `memory_embeddings` 를 비워 두고 키워드 검색만 씁니다.

| 테이블 | 한 행이 뜻하는 것 | 왜 따로 두나 |
| --- | --- | --- |
| `memories` | 기억 한 건(5줄 이내 요약) | 검색·권한·만료의 기준 |
| `memory_sources` | 기억 한 건의 근거 하나(파일·커밋·카드·URL) | 기억 하나에 근거가 여러 개 붙음. 근거 파일이 바뀌면 `ref` 로 관련 기억을 역추적 |
| `memory_embeddings` | 기억 한 건을 특정 모델로 만든 벡터 하나 | 모델을 바꾸면 차원이 달라짐. 새 모델 벡터를 옆에 쌓고 옛 모델 행만 지우면 됨 |

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;    -- 한국어 부분 일치 검색
CREATE EXTENSION IF NOT EXISTS vector;     -- MEMORY_EMBED=local 일 때 사용(설치만 해 둠)

-- ① 본문
CREATE TABLE memories (
  id          bigserial PRIMARY KEY,
  namespace   text NOT NULL DEFAULT 'default',   -- 프로젝트 구분(OFFICE_ID_PREFIXES 의 구분과 맞춤)
  type        text NOT NULL CHECK (type IN ('episode','fact','decision','procedure')),
  card_id     text,                              -- TASK-XXX-001 등, 없으면 NULL
  lane        text CHECK (lane IN ('fe','be','office','orch')),
  role        text NOT NULL,                     -- 쓴 역할(FLEET_ROLE)
  session_id  text,                              -- 헤드리스 로그와 맞추기 위한 값(agent-run 로그 파일명)
  content     text NOT NULL CHECK (length(content) <= 2000),
  tags        text[] NOT NULL DEFAULT '{}',
  supersedes  bigint REFERENCES memories(id),    -- 이 행이 대체한 옛 기억
  valid_until timestamptz,                       -- episode 는 MEMORY_TTL_DAYS, 나머지는 NULL
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX memories_content_trgm ON memories USING gin (content gin_trgm_ops);
CREATE INDEX memories_tags         ON memories USING gin (tags);
CREATE INDEX memories_card         ON memories (namespace, card_id);
CREATE INDEX memories_type_time    ON memories (namespace, type, created_at DESC);
CREATE INDEX memories_supersedes   ON memories (supersedes);

-- ② 근거
CREATE TABLE memory_sources (
  id          bigserial PRIMARY KEY,
  memory_id   bigint NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
  kind        text NOT NULL CHECK (kind IN ('file','commit','card','adr','url')),
  ref         text NOT NULL,                     -- 경로 · 커밋 해시 · 카드 ID · URL
  line_from   int,                               -- 파일 근거일 때 행 범위(선택)
  line_to     int,
  content_hash text,                             -- 적재 시점 근거 내용의 해시 — 파일이 바뀌었는지 판정
  UNIQUE (memory_id, kind, ref)
);
CREATE INDEX memory_sources_ref ON memory_sources (kind, ref);
-- memory-sync.sh 의 멱등성: 같은 (kind, ref, content_hash) 가 이미 있으면 새 기억을 만들지 않음
CREATE UNIQUE INDEX memory_sources_sync ON memory_sources (kind, ref, content_hash) WHERE content_hash IS NOT NULL;

-- ③ 벡터 (처음에는 비워 둠)
CREATE TABLE memory_embeddings (
  memory_id   bigint NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
  model       text NOT NULL,                     -- MEMORY_EMBED_MODEL 값 그대로
  dim         int NOT NULL,
  embedding   vector NOT NULL,                   -- 차원 제한 없는 열. 인덱스는 모델별 부분 인덱스로
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (memory_id, model)
);
-- MEMORY_EMBED=local 로 켤 때 모델마다 한 번 만듭니다(차원 N 은 모델에 맞춰 — [확인 필요: 모델 차원]).
-- CREATE INDEX memory_embeddings_<모델약칭> ON memory_embeddings
--   USING hnsw ((embedding::vector(N)) vector_cosine_ops) WHERE model = '<모델>';

-- 검색에서 늘 쓰는 "살아 있는 기억" 뷰
CREATE VIEW live_memories AS
SELECT m.* FROM memories m
WHERE (m.valid_until IS NULL OR m.valid_until > now())
  AND NOT EXISTS (SELECT 1 FROM memories n WHERE n.supersedes = m.id);
```

**지우지 않습니다.** 틀린 기억은 새 행을 쓰고 `supersedes` 로 가립니다. `ON DELETE CASCADE` 는 `memory-gc.sh` 가 사람 승인 후 오래된 episode 를 실제로 비울 때만 쓰입니다.

**근거가 바뀌면**: `memory-sync.sh` 가 파일의 현재 해시와 `memory_sources.content_hash` 를 비교해, 다르면 그 기억의 `tags` 에 `stale` 을 붙이고 검색 결과에 "근거 변경됨" 표시를 함께 돌려줍니다. 새 내용은 새 기억으로 적재하고 옛 것을 `supersedes` 로 가립니다.

**나중에 붙일 수 있는 테이블**(지금은 만들지 않음): `memory_access_log`(어떤 기억이 실제로 쓰였나 — 1~2주 운영 뒤), `entities`·`relations`(카드 의존·시스템 관계 질문이 늘 때).

#### (4) 접근 창구 — MCP 서버

- 도구는 두 개만 엽니다. `memory_search(query, card_id?, type?, tags?, k?)` 는 `live_memories` 뷰에서 찾고(만료·대체된 기억 제외), 키워드 점수(+ 켜져 있으면 현재 `MEMORY_EMBED_MODEL` 의 `memory_embeddings` 유사도)로 상위 k 개를 **`memory_sources` 근거와 함께** 돌려줍니다. `stale` 태그가 붙은 기억은 "근거 변경됨" 을 표시합니다. `memory_write(type, content, card_id?, tags?, sources?)` 는 `memories` 1행과 `memory_sources` n행을 한 트랜잭션으로 쓰고(임베딩이 켜져 있으면 `memory_embeddings` 도), 쓰기 전에 검사 세 가지를 합니다 — ① `FLEET_ROLE` 이 그 type 을 쓸 권한이 있는지 ② `OFFICE_SECRET_PATTERNS` 에 걸리는 값(키·비밀번호·주민번호·계좌)이 있는지(있으면 거부) ③ 같은 카드에 거의 같은 내용이 있는지(있으면 새로 쓰지 않고 기존 id 반환).
- 삭제·수정 도구는 열지 않습니다. 정리는 `memory-gc.sh` 가 합니다.
- 세션에 붙이는 방법: 헤드리스는 `agent-run.sh` 가 `--mcp-config $FLEET/.fleet/mcp/memory.json` 으로, 수동 탭은 `wt-new.sh` 가 인쇄하는 시작 명령에 같은 옵션을 넣습니다. worktree 에 `.mcp.json` 을 두지 않으므로 코드 저장소에 섞일 일이 없습니다.
- 접속 문자열(`MEMORY_DB_URL`)은 MCP 설정 파일에 직접 쓰지 않고 환경 변수로 넘기며, `Read(**/.fleet/mcp/*.env)` 를 deny 에 넣습니다.

#### (5) 언제 읽고 언제 쓰나

| 시점 | 누가 | 무엇을 |
| --- | --- | --- |
| 카드 착수 | DEV · OFFICE | `memory_search` — 카드 ID · scope 경로 · 핵심 키워드. 결과는 참고용이고, 파일과 다르면 파일을 따름 |
| `*-hard` 호출 전 | DEV | 같은 오류 문구로 `memory_search(type=procedure)` — 이미 풀어 본 문제인지 |
| 완료 보고 직전 | DEV · OFFICE | `memory_write(type=episode)` 5줄 이내 |
| 세션 중단 인계(4.4) | 인계 에이전트 | 인계 문서 작성 후 `memory-sync.sh` 로 적재 |
| 통합 직후 | INTEGRATOR | 카드에서 확인된 사실만 `fact` 로 |
| 하루 끝 | 루프 | `memory-sync.sh`(ADR·인계 문서) → `memory-gc.sh`(만료 표시·중복 병합) → `memory-up.sh backup` |

#### (6) 서브에이전트와의 관계

서브에이전트는 MCP 도구를 상위 세션에서 물려받지 못할 수 있습니다 `[확인 필요: 설치된 Claude Code 버전에서 서브에이전트의 MCP 도구 상속 여부]`. 그래서 **기억 조회·기록은 최상위 DEV·OFFICE 세션만** 하고, `*-explore` 등에는 조회 결과를 프롬프트로 넘깁니다.

**Claude 에게 줄 지시문 ⑦**

```
장기기억 DB 를 구성하십시오. 모든 값은 .fleet/config.env 의 MEMORY_* 를 읽고, 경로·포트·계정을 하드코딩하지 마십시오.
1) .fleet/memory/docker-compose.yml: postgres(pgvector 확장이 포함된 공식 이미지 [확인 필요: 이미지 태그]) 1개, 127.0.0.1:$MEMORY_PG_PORT 로만 바인드, 데이터는 이름 있는 볼륨. MEMORY_DB_URL 이 비어 있으면 비밀번호를 생성해 config.env 에 채우고 출력하지 마십시오.
2) .fleet/memory/schema.sql: 2-7 (3) 의 테이블 3개 + live_memories 뷰 그대로. memory_embeddings 는 만들되 HNSW 인덱스는 MEMORY_EMBED=local 로 켤 때 scripts/memory-embed-index.sh <모델> <차원> 이 만들게 하십시오. 스키마 변경은 .fleet/memory/migrations/NNN_*.sql 로 번호를 붙여 쌓고, 적용 이력을 schema_migrations 테이블에 남기십시오.
3) scripts/memory-up.sh up|down|status|backup: status 는 접속과 memories 건수를 출력, backup 은 pg_dump 를 .fleet/backup/memory-<시각>.sql.gz 로.
4) .fleet/memory/server/: memory_search · memory_write 두 도구만 가진 stdio MCP 서버(Python). 2-7 (4) 의 검사 세 가지를 구현하고, MEMORY_EMBED=local 일 때만 임베딩을 계산합니다. 의존성은 $FLEET/.venv 에 설치합니다(관제 저장소 venv — 코드 저장소 requirements.txt 는 건드리지 않음).
5) .fleet/mcp/memory.json: 위 서버를 띄우는 MCP 설정. 접속 문자열은 환경 변수 참조로만.
6) scripts/memory-sync.sh: .fleet/adr/*.md → decision(kind=adr), .fleet/daily/*_인계.md → episode(kind=file) 로 적재. memory_sources 의 (kind, ref, content_hash) 로 멱등. 기존 근거의 해시가 바뀌었으면 옛 기억에 stale 태그 + 새 기억을 supersedes 로.
7) scripts/memory-gc.sh: 같은 card_id 에서 내용이 거의 같은 것은 supersedes 로 묶기(만료는 live_memories 뷰가 이미 거름). 실제 삭제는 --purge <일수> 를 사람이 줄 때만, 만료 후 그 일수가 지난 episode 에 한해 합니다(근거·벡터는 CASCADE 로 함께 삭제).
8) agent-run.sh 와 wt-new.sh 에 2-3 규격 9번을 반영하고, settings.local.json deny 에 Read(**/.fleet/mcp/*.env) 를 추가하십시오.
9) 시험: memory-up.sh up → status → memory-sync.sh → 시험 카드 하나로 agent-run 을 돌려 로그에 memory_search·memory_write 호출이 있는지, DB 에 episode 1건과 그 근거 행(memory_sources)이 생겼는지 확인. ADR 파일 하나를 고친 뒤 memory-sync.sh 를 다시 돌려 옛 기억에 stale 이 붙고 새 기억이 supersedes 로 연결되는지도 확인. 이어서 비밀번호 형태의 문자열로 memory_write 를 시도해 거부되는지, dev 역할로 type=fact 쓰기가 거부되는지 확인. 마지막으로 free -h 로 DB 를 띄운 뒤의 available 을 보고하십시오.
10) 전부 통과하면 MEMORY_ENABLE=1 로 바꾸고 결과를 두괄식으로 보고하십시오.
```

* * *

## 3. 모델 배분과 위임 규칙 (루트 CLAUDE.md 에 넣을 절)

```markdown
## 10. 모델 배분과 위임 (토큰 절약 v2)

- 판단은 Fable, 구현은 Opus(문서·디자인은 Opus 1M), 정해진 명령·잡일은 Haiku 입니다. 모델은 `.fleet/config.env` 의 `MODEL_*` 로 정하고 스크립트가 `--model` 로 넘깁니다. 세션 안에서 바꾸지 않습니다.
- 구현 에이전트는 **탐색·검증 로그 읽기·요약을 직접 하지 않고** 자기 worktree 의 `.claude/agents/` 서브에이전트(`*-explore` `*-verify` `*-summarize`, 문서는 `doc-find` `doc-check`)에 맡깁니다.
- **Fable 은 아래 조건에서 반드시 불립니다**(`*-hard` · `doc-judge`): 같은 카드에서 2회 실패 · RF 카드 · 티어 B 카드 · ADR 필요 · 변경 5개 파일 초과 예상 · 원인 불명 · 문서 초안 완성 및 수정 반영 직후. 일반 구현에는 부르지 않습니다.
- **오케스트레이터가 Agent 도구로 서브에이전트를 띄울 때는 `model` 인수를 반드시 명시합니다.** 빼면 상위 모델이 상속되어 구현이 Fable 로 돌아갑니다. 구현·인계는 `opus`, 잡일은 `haiku`, `*-hard` 조건일 때만 `fable` 입니다. worktree 의 `settings.local.json` 의 model 은 그 worktree 에서 `claude` 를 새로 열 때만 적용되고 Agent 도구 호출에는 적용되지 않습니다.
- 완료 보고 마지막 줄에 서브에이전트 호출 횟수를 적습니다. 카드당 `*-hard` 1~2회 · `doc-judge` 2~3회가 정상 범위이고, 카드의 30% 이상에서 `*-hard` 가 불리면 카드 명세를 더 구체적으로 쓰는 쪽으로 고칩니다.

## 11. 커밋 계정과 장기기억 (v4)

- 커밋 계정은 `.fleet/config.env` 의 `COMMIT_AUTHOR_*` 로 고정돼 있습니다. `git config` 를 바꾸거나 `--author` 를 쓰지 않고, 커밋 메시지에 공동 작성자 줄을 넣지 않습니다. `check-identity.sh` 를 통과하지 못한 카드는 리뷰·통합되지 않습니다.
- `MEMORY_ENABLE=1` 이면 착수 전에 `memory_search`, 완료 보고 직전에 `memory_write(type=episode)` 를 합니다. 기억은 참고용이며 카드·ADR·보드 파일과 다르면 파일을 따릅니다. 비밀 값은 기억에 쓰지 않습니다.
```

* * *

## 4. 운영 방법

### 4.1 수동 (탭 여러 개)

`wt-new.sh` 가 인쇄하는 `cd … && claude` 명령을 새 탭에 붙입니다. `settings.local.json` 의 model 이 적용되므로 `--model` 을 따로 줄 필요가 없습니다. 오케스트레이터 탭만 `claude --model claude-fable-5-1` 로 엽니다.

### 4.2 자율 운행

```bash
cd ~/work/<프로젝트>-agents
nohup scripts/fleet-loop.sh > .fleet/logs/loop.out 2>&1 &   # 시작
scripts/fleet-tell.sh "status"                               # 보고서 즉시 갱신
scripts/fleet-tell.sh "approve TASK-XXX-001"                 # 티어 B/C 통합 승인
scripts/fleet-stop.sh                                        # 정지
```

### 4.3 에이전트가 멈췄을 때 (시계로 판정합니다)

에이전트가 "알림을 기다린다" 며 멈추는 것은 진행이 아닙니다. 보고 문장을 믿지 말고 **실제 진척**(최신 커밋 해시 · `.fleet/logs/<ID>/verify-*` 생성 여부 · `lock.sh status` · 포트 리스너)을 확인합니다.

| 시점 | 조치 |
| --- | --- |
| 대기 보고 즉시 | "결과를 직접 확인하고 기다리지 말고 진행하라" 를 10분 기한과 함께 보냅니다 |
| 10분 경과, 새 커밋·verify 로그 없음 | 에이전트를 세우고 인계 에이전트를 새로 띄웁니다. 미커밋 작업은 인계 에이전트가 먼저 커밋합니다 |
| 멈춘 카드가 핫스팟 락 보유 | 인계 에이전트의 첫 일은 커밋 → `lock.sh release` 입니다 |
| 교체 후 | 카드 프런트매터에 `handover: YYYY-MM-DD HH:MM <사유>` 한 줄. 착수 시각은 원래 worktree 생성 시각을 유지합니다 |

핫스팟 락은 **획득 → 편집 → 커밋 → 해제**를 10분 안에 끝냅니다. 락을 쥔 채 화면 확인이나 QA 를 하지 않습니다.

### 4.4 세션이 죽었을 때 — 작업물 보존 절차

헤드리스 세션은 메모리 부족 · 타임아웃 · 모델 한도로 예고 없이 끊깁니다. **끊긴 세션의 작업물은 커밋되지 않은 채 worktree 에 남아 있습니다.** 버리지 말고 아래 순서로 보존합니다. 순서를 지키면 다음 세션이 맥락을 다시 쌓지 않아도 이어갈 수 있습니다.

1. **사실 확인** — 보고 문장이 아니라 상태로 판정합니다.
   ```bash
   free -h                                   # 메모리가 원인인지
   ps -eo pid,rss,args | grep "[c]laude"     # 살아 있는 세션
   docker ps                                 # 떠 있는 컨테이너
   for w in wt/*/; do echo "== $w"; git -C "$w" status --short; git -C "$w" log --oneline -1; done
   ```
2. **작업물 커밋** — 카드마다 미커밋 변경을 그대로 커밋합니다. 메시지에 접두사(`Feat:`/`Fix:`/`Chor:`/`Refactor:`)와 카드 ID 를 넣고, 본문에 **"진행 중 · verify 미실행"** 과 중단 시각·사유를 적습니다. 미완이어도 커밋합니다 — 브랜치는 통합 지점이 아니므로 안전하고, 남겨 두면 다음 세션이 자기 변경과 구분하지 못합니다.
3. **카드에 기록** — 프런트매터에 `started:`(원래 착수 시각 유지)와 `handover: YYYY-MM-DD HH:MM <사유>` 한 줄. 상태 JSON 의 `state` 를 `in-progress` 로 맞추고 `board.sh` 를 돌립니다.
4. **인계 문서** — `.fleet/daily/<날짜>_인계.md` 에 카드별로 **한 일 · 커밋 해시 · 남은 단계**를 적습니다. 다음 세션은 이 문서부터 읽습니다.
5. **원인 제거 후 재기동** — 메모리가 원인이면 상한을 낮춰 동시 2장 이하로 다시 띄웁니다. 같은 수로 재시도하면 같은 자리에서 다시 죽습니다.

착수 조건이 미충족인 카드(외부 계정·키가 필요한 카드)는 이 기회에 `wt-rm.sh` 로 되돌려 자리를 비웁니다. 브랜치는 남으므로 조건이 갖춰지면 `wt-new.sh` 로 다시 띄우면 됩니다.

### 4.5 반영 후 1~2일 점검 항목

| 지표 | 보는 곳 | 정상 범위 | 벗어나면 |
| --- | --- | --- | --- |
| fe verify 소요 | `.fleet/logs/<ID>/verify-fe.*` | ext4 기준 수십 초 | 분 단위면 9p 경로·빌드 폴백을 의심 |
| 카드당 dev 재기동 횟수 | `.fleet/run/*.tries.dev`, `loop.log` | 평균 1.3~1.5 | 2 이상이면 카드 명세·explore 보고 상세도 상향 |
| `changes-requested` 비율 | `.fleet/reviews/*.md` | v1 대비 +10%p 이내 | 설계 규율 문구를 dev 역할 파일에 강화 |
| `*-hard` 호출 비율 | 완료 보고 마지막 줄 | 카드의 30% 이하 | 명세 상향(Fable 을 더 부르는 쪽이 아님) |
| `*-hard` 미호출인데 3회 실패 | `awaiting-human` 사유 | 0건 | `HARD_AFTER_TRIES` 를 2로 |
| 문서 `doc-judge` 호출 | 완료 보고 | 카드당 2~3회 | 0회면 office 역할 파일 규칙 확인 |
| 레인 상한에 걸린 카드 | `wt-new.sh` 거부 메시지 | 0건 | `merged` 로 멈춘 카드를 `wt-rm.sh` 로 `archived` 처리 |
| (v4) 계정 위반 커밋 | `check-identity.sh` 결과 | 0건 | 반복되면 settings `env`·공동 작성자 설정 키가 버전과 맞는지 확인 |
| (v4) 기억 조회 후 재작업 감소 | 카드당 dev 재기동 횟수(도입 전후 비교) | 도입 전보다 낮음 | 변화 없으면 `memory_write` 내용이 너무 짧거나 검색어가 카드와 안 맞는 것 |
| (v4) 기억 건수 증가 | `memory-up.sh status` | 카드 1장당 1~3건 | 급증하면 중복 검사·gc 확인 |

* * *

## 5. 자주 막히는 곳

- **모델 ID 오류로 기동 실패**: `claude --model claude-fable-5-1 -p "hi"` 로 단독 확인. 계정에 Fable 이 없으면 `MODEL_ORCH` `MODEL_HARD` 를 `opus` 로 내리고 나머지는 그대로 둡니다.
- **서브에이전트가 안 불림**: description 에 "반드시 호출하십시오" 조건이 있는지, worktree 의 `.claude/agents/` 에 복사됐는지 확인합니다. 서브에이전트는 서브에이전트를 못 부르므로 DEV 세션이 직접 불러야 합니다.
- **Opus 1M 이 비싸짐**: 20만 토큰을 넘는 컨텍스트는 단가가 오릅니다. 문서·디자인 외에는 `opus` 를 씁니다.
- **verify 가 느림**: 경로가 `/mnt/` 인지부터 봅니다(`stat -f -c %T .` 가 `ext2/ext3` 계열이면 정상, `v9fs` 면 9p). 0.1 참조.
- **PDF 가 저품질로 나옴**: 변환 스크립트에 `[[ "$OUT" == /mnt/c/* ]]` 같은 경로 조건이 남아 있으면 ext4 산출물이 조용히 fpdf2 폴백으로 떨어집니다. 출력 문구가 `pdf 생성(chrome headless)` 인지로 판정합니다.
- **marp 가 윈도우 Chrome 으로 실패**: DevTools 포트가 방화벽에 막히는 환경이 있습니다. WSL 안의 리눅스 Chrome(puppeteer 캐시)을 먼저 시도하는 경로를 두면 해결됩니다(`pptx 생성(marp, 리눅스 Chrome)`).
- **be verify 가 1단계에서 멈춤**: Docker Desktop 이 꺼져 있으면 `/usr/bin/docker` 심볼릭 링크가 사라집니다.
- **레인 상한에 걸려 worktree 생성 거부**: `active_ids_for_lane()` 이 `state != "archived"` 를 전부 활성으로 세므로, 통합 후 `merged` 로 멈춘 카드가 자리를 차지합니다. `wt-rm.sh` 로 정리합니다(브랜치는 유지됩니다).
- **헤드리스 세션이 한꺼번에 죽음**: 거의 항상 메모리입니다. `free -h` 의 available 과 스왑을 봅니다. WSL 기본값(호스트 50%)에서 세션 6개 + 컨테이너 11개를 돌리면 8GB 급에서는 확실히 죽습니다. 0-0 (1) 로 상한을 다시 잡고, `.wslconfig` 로 메모리를 올린 뒤 `wsl --shutdown` 합니다. 죽은 뒤의 수습은 4.4 입니다.
- **verify 를 돌렸더니 완료된 카드가 다시 진행 중으로 보임**: `verify.sh` 가 상태를 씁니다. 점검 목적이면 `verify-fe.sh <ID> <WT> <LOGDIR>` 로 직접 부릅니다.
- **worktree 를 더 못 만듦(레인 상한)**: 통합이 끝났는데 `archived` 가 아닌 카드가 자리를 차지하고 있습니다. `wt-rm.sh` 로 정리하면 브랜치는 남고 자리만 돌아옵니다.
- **`next build` 가 `Symlink node_modules is invalid, it points out of the filesystem root`**: worktree 의 `node_modules` 를 통합본 것으로 링크했을 때 Turbopack 의 workspace root 가 worktree 안쪽이라 링크가 밖을 가리키는 것으로 보입니다. 부록 C 참조.
- **`claude -p` 가 600초 만에 끝나고 검증 결과가 없음**: 백그라운드로 돌린 verify 를 기다리다 잘린 것입니다. 포그라운드 + `timeout` 으로 돌립니다(2-3 규격 6번).
- **Alembic heads 2개**: 락을 잡지 않고 두 카드가 동시에 리비전을 만든 경우입니다. 카드에 `schema_change: true` 가 없으면 락이 거부되도록 `lock.sh` 를 짭니다.
- **(v4) 원격 화면에서 커밋이 여러 사람으로 보임**: 이메일이 원격 계정에 등록된 것과 다르거나, 공동 작성자 줄이 붙은 경우입니다. `git log --format='%an <%ae> | %cn <%ce>'` 와 메시지 끝 줄을 봅니다. 이미 원격에 올라간 이력은 고치지 말고 이후 커밋부터 맞춥니다.
- **(v4) `check-identity.sh` 가 머지 커밋에서 실패**: `integrate.sh` 를 사람 셸에서 `_common.sh` 없이 돌린 경우입니다. 스크립트로만 통합합니다.
- **(v4) 에이전트가 기억을 안 찾음**: `agent-run.sh` 로그 첫 줄에 `--mcp-config` 가 붙었는지, `memory-up.sh status` 가 정상인지 봅니다. DB 가 내려가면 옵션이 빠진 채 조용히 진행됩니다(설계상).
- **(v4) 오래된 기억이 잘못된 판단을 부름**: `fact` 가 바뀌었는데 이전 기억이 살아 있는 경우입니다. 오케스트레이터가 새 `fact` 를 `supersedes` 로 쓰고, 근거 파일을 먼저 고칩니다.
- **`pkill -f` 로 셸이 죽음**: `pgrep -f` · `pkill -f` 는 자기 셸 명령줄까지 잡습니다. `fuser -k <포트>/tcp` 뒤에 `ps -eo pid,args | grep "next dev -p 310[N]" | grep -v grep` 으로 pid 를 뽑아 kill 합니다.

* * *

## 6. 이 가이드로 만들어지는 것 (체크리스트)

- [ ] **0-0 사전 준비**: `.wslconfig` 로 정한 메모리(`free -h` 로 확인) · 도구 6종 · `claude` 모델 ID 3개 기동 확인 · 저장소가 `/home` 아래
- [ ] **작업 경로가 전부 ext4(`/home/...`)** · 윈도우는 볼트(읽기) · 산출물함(쓰기) · Chrome(변환) 세 지점만
- [ ] **동시 상한이 메모리에서 산정됨**: `MAX_WT_FE`·`MAX_WT_BE`·`MAX_CONCURRENT_AGENTS` 가 0-0 (1) 산정식과 맞음
- [ ] **최초 세팅(2-0)**: 기준선 태그 `fleet-baseline-*` + `.fleet/backup/*.bundle` · `integration` 브랜치와 `wt/integration` · `.git/info/exclude` 6항목 · `<BE_DIR>/.env`(비밀 키) · 사무 저장소 2개 · `OUTBOX` 폴더
- [ ] 보호 규칙 확인: 개발 브랜치·사무 `main` 직접 커밋 없음 · push 는 사람 · `LOOP_AUTO_PUSH=0` 으로 시작
- [ ] `CLAUDE.md`(공통 규칙 + 10절 모델 배분·위임) · `README.md`
- [ ] `.fleet/config.env`(경로 7개 · `MODEL_*` 7개 · `HARD_AFTER_TRIES` · `OUTBOX`)
- [ ] `scripts/` 20종 이상(2-2 목록) — `agent-run.sh` 에 레인별 `--model` 과 `*-hard` 강제 조건, `wt-new.sh` 에 settings model 과 agents 복사, **경로 하드코딩 0건**
- [ ] `roles/` 8종(dev-fe·dev-be·office 에 위임 규칙 블록)
- [ ] `roles/agents/fe/` 4 · `be/` 4 · `office/` 3 = 서브에이전트 11개
- [ ] `~/.claude/agents/` fleet-reviewer(opus) · fleet-qa · fleet-integrator(sonnet)
- [ ] 시험 카드 `TASK-SMOKE-001` 로 model·agents·hard 강제 문구·verify 소요·**메모리 여유(available ≥ 2GB)** 확인 후 제거
- [ ] `agent-run.sh` 프롬프트에 "검증은 포그라운드로" 문장이 있고, 동시 실행 상한을 지킴
- [ ] **(v4) 커밋 계정**: `COMMIT_AUTHOR_*` 가 config 에 있고, 저장소 4곳 `git config user.email` 일치 · worktree settings `env` · 공동 작성자 줄 차단 · `check-identity.sh` 가 review-req·integrate·publish 에 연결 · 시험에서 위반 커밋 거부 확인
- [ ] **(v4 · 선택) 장기기억 DB**: `memory-up.sh status` 정상 · MCP 도구 2개 · 비밀 값·권한 없는 fact 쓰기 거부 확인 · `memory-sync.sh`·`memory-gc.sh`·백업이 하루 끝 루프에 들어감 · `MEMORY_ENABLE=1`
- [ ] 사무 레인을 쓴다면 시험 발행 1회로 `pdf 생성(chrome headless)` · `pptx 생성(marp…)` · `OUTBOX` 사본 확인

* * *

## 부록 A. 이미 `/mnt/c` 에 구축했다면 — 이전 절차

결론: **전부 옮깁니다.** fleet · 메인 저장소 · 사무 저장소 2개를 `/home/<사용자>/work/` 로 복사하고, 발행 산출물만 `C:\work\산출물\` 로 자동 복사하게 바꿉니다. 원본을 지우는 것은 검증을 전부 통과한 뒤 마지막이므로 언제든 되돌릴 수 있습니다. 서화에서는 **약 3.5G · 29분**(복사 16분 + 치환·복구 3분 + 검증 10분)이 걸렸고 검증 11항목 전부 통과했습니다.

부분 이전(fleet 만, 또는 코드 저장소만)은 권하지 않습니다. 경계를 넘는 조합(`/home` 의 worktree ↔ `/mnt/c` 의 부모 저장소)이 생겨 오히려 복잡해집니다.

### A-0. 옮기는 대상

| 대상 | 전 | 후 |
| --- | --- | --- |
| fleet | `/mnt/c/work/<프로젝트>-agents` | `/home/<사용자>/work/<프로젝트>-agents` |
| 메인 저장소 | `/mnt/c/work/<프로젝트>` | `/home/<사용자>/work/<프로젝트>` |
| 사무 저장소 2개 | `/mnt/c/work/<프로젝트>-office` · `/mnt/c/work/office` | `/home/<사용자>/work/…` |
| 산출물 사본 | (저장소 안에만 있었음) | **`C:\work\산출물\<구분>\<ID>\`** (새로 생김) |
| Obsidian 볼트 | `/mnt/c/Users/<이름>/Obsidian/업무` | **그대로** |

### A-1. 먼저 고칠 코드 2곳

**① 변환 스크립트의 경로 조건 제거** — PDF 가 조용히 저품질 폴백으로 떨어지는 것을 막습니다.

```bash
# 전
if [[ ! -s "$PDFOUT" ]] && command -v wslpath >/dev/null 2>&1 && [[ "$OUT" == /mnt/c/* ]]; then
# 후
if [[ ! -s "$PDFOUT" ]] && command -v wslpath >/dev/null 2>&1; then
```

**② 발행 스크립트에 산출물함 복사 단계 추가** — 산출물 커밋 다음, 완료 인쇄 앞에 넣습니다.

```bash
# ⑤ 윈도우 산출물함으로 복사 (한컴·엑셀로 여는 곳)
WINOUT="$OUTBOX/$(outbox_dir_of "$PROJECT")/$ID"
if mkdir -p "$WINOUT" && cp -f "$OUTDIR"/* "$WINOUT"/; then
    info "윈도우 산출물함 복사: $WINOUT"
else
    warn "윈도우 산출물함 복사 실패 — 저장소 산출물은 정상입니다: $OUTDIR"
fi
```

복사 실패가 발행을 되돌리지 않게 합니다. 정본은 저장소 안 `산출물/<ID>/` 이고 `C:\work\산출물\` 은 사본입니다.

### A-2. 단계

**0단계 — 사전 정리**: 진행 중 카드 0장 · 락 0건 · 미커밋 파일 처리 방침 확인.

**1단계 — 정지**

```bash
docker compose -p <프로젝트>-integration down
ps -eo pid,args | grep "next dev -p 3" | grep -v grep      # 있으면 부모 pid 를 kill
ss -ltnp | grep -E ':(3000|3101|3102|3103|8100|8101)'      # 비어 있어야 합니다
```

`pkill -f` · `pgrep -f` 는 자기 셸 명령줄까지 잡아 셸이 죽습니다. 쓰지 않습니다.

**2단계 — 복사(원본 유지)**

```bash
mkdir -p ~/work
for d in <프로젝트>-agents <프로젝트> <프로젝트>-office office; do
    rsync -a --info=progress2 "/mnt/c/work/$d/" "$HOME/work/$d/"
done
# 검증 — 네 쌍의 파일 수가 같아야 합니다
for d in <프로젝트>-agents <프로젝트> <프로젝트>-office office; do
    printf "%-20s %7s %7s\n" "$d" \
        "$(find /mnt/c/work/$d -type f | wc -l)" "$(find $HOME/work/$d -type f | wc -l)"
done
```

`rsync -a` 는 심볼릭 링크를 링크인 채로 복사합니다(내용이 아니라 링크이므로 4단계에서 다시 겁니다).

**3단계 — 경로 치환**

```bash
cd ~/work/<프로젝트>-agents
grep -rl "/mnt/c/work/" --include='*.sh' --include='*.env' --include='*.json' \
     --include='*.md' --include='*.py' --include='*.mjs' \
     scripts/ office-tools/ roles/ .fleet/config.env wt/*/.claude/ 2>/dev/null \
  | xargs perl -i -pe "s{/mnt/c/work/}{$HOME/work/}g"
```

**`/mnt/c/work/` 아래만 바꿉니다.** 아래는 `work` 밑이 아니므로 자동으로 안전하지만, 치환 규칙을 임의로 넓히면 변환이 깨집니다.

| 경로 | 결과 | 왜 지켜야 하는가 |
| --- | --- | --- |
| `/mnt/c/Program Files/Google/Chrome/…` | 그대로 | 건드리면 PDF·PPTX 변환이 깨집니다 |
| `/mnt/c/Program Files (x86)/Microsoft/Edge/…` | 그대로 | 같음 |
| `/mnt/c/Windows/Fonts/malgun.ttf` | 그대로 | fpdf2 폴백의 한글 폰트 |
| `/mnt/c/Users/<이름>/Obsidian/업무` | 그대로 | 볼트 |
| `/mnt/c/work/산출물` (새 `OUTBOX`) | **치환에 걸립니다 — 주의** | `OUTBOX` 줄은 **치환이 끝난 뒤에** 추가합니다 |

검증:

```bash
grep -rn "/mnt/c/Program Files\|/mnt/c/Windows" office-tools/   # 살아 있어야 합니다
grep -n "VAULT" .fleet/config.env                                # 볼트 경로 살아 있어야 합니다
grep -rn "/mnt/c/work/" scripts/ office-tools/ roles/ .fleet/config.env wt/*/.claude/   # 0건
```

**4단계 — 코드 수정 · 고리 복구**

```bash
echo "OUTBOX=/mnt/c/work/산출물" >> ~/work/<프로젝트>-agents/.fleet/config.env
mkdir -p '/mnt/c/work/산출물/<구분1>' '/mnt/c/work/산출물/<구분2>'
```

worktree 양방향 링크를 복구합니다. **`git worktree repair` 에 의존하지 마십시오** — 서화에서는 repair 가 worktree 의 `.git` 파일을 따라가 **옛 부모(`/mnt/c`)의 gitdir 을** 고쳐 버려서 결과가 "새 worktree ↔ 옛 부모" 가 됐고 `worktree list` 가 계속 옛 경로를 보여줬습니다. 양방향 두 파일을 직접 쓰는 편이 빠릅니다.

```bash
# worktree 쪽: <worktree>/.git 파일 한 줄
gitdir: /home/<사용자>/work/<프로젝트>/.git/worktrees/<이름>
# 부모 쪽: <부모>/.git/worktrees/<이름>/gitdir 파일 한 줄
/home/<사용자>/work/<프로젝트>-agents/wt/<이름>/.git
# 확인
git -C ~/work/<프로젝트> worktree list          # 전부 /home
git -C ~/work/<프로젝트>-office worktree list   # 전부 /home
```

`node_modules` 가 심볼릭 링크였다면 새 경로로 다시 겁니다(ext4 에서는 `npm ci` 실물 설치로 바꾸는 편이 낫습니다).

**5단계 — 세션 설정**: `~/.claude` 의 프로젝트별 설정·권한 허용 목록에 옛 경로가 남았는지 확인하고, 새 경로에서 `claude` 를 열어 worktree 의 `settings.local.json` 훅이 도는지 봅니다.

**6단계 — 검증 (전부 통과해야 7단계로 갑니다)**

| # | 항목 | 통과 기준 |
| --- | --- | --- |
| 1 | `scripts/board.sh` | 갱신되고 작업 건수가 이전과 같음 |
| 2·3 | 양쪽 저장소 `worktree list` | 전부 새 경로 |
| 4 | 각 worktree `git status` | 브랜치·미커밋 상태가 이전과 동일 |
| 5 | `local-stack.sh up` | 프론트·백엔드가 뜨고 화면이 열림 |
| 6 | fe verify 1회 | 통과. **소요 시간을 이전 값과 비교해 기록** |
| 7 | be verify 1회 | 통과 (Docker Desktop 필요) |
| 8 | 사무 발행 1회(docx+pdf) | 산출물함에 사본이 생기고 출력이 `pdf 생성(chrome headless)` |
| 9 | 사무 발행 1회(slides) | `pptx 생성(marp…)` (python-pptx 폴백이 아님) |
| 10 | 경로 가드 | worktree 밖 파일 수정 시도가 exit 2 로 차단 |
| 11 | Turbopack | 빌드가 webpack 재빌드로 떨어지지 않음 |

8·9 는 **출력 문구로** 판정합니다. `fpdf2 폴백` · `python-pptx 폴백` 이면 실패로 봅니다.

**7단계 — 원본 정리(6단계 전부 통과 후)**

```bash
for d in <프로젝트>-agents <프로젝트> <프로젝트>-office office; do
    mv "/mnt/c/work/$d" "/mnt/c/work/$d.old"
done
```

하루 이상 쓰면서 문제가 없으면 `.old` 를 지웁니다. 바로 지우지 않습니다.

**롤백**: 6단계 어디서든 실패하면 새 경로를 지우면 됩니다. C 쪽 `.git` 을 건드리지 않았으므로 복구할 것이 없습니다.

### A-3. 실제로 겪은 함정 7가지

1. **`OUTBOX` 를 치환 전에 추가** — `/mnt/c/work/산출물` 이 함께 바뀝니다. 4단계에서 추가합니다.
2. **치환 범위를 넓히는 것** — `/mnt/c/Program Files` · `/mnt/c/Windows` 가 바뀌면 PDF·PPTX 가 깨집니다.
3. **`settings.local.json` 훅 경로 누락** — 경로 가드가 **조용히** 무력화됩니다. 오류가 나지 않으므로 직접 확인해야 합니다.
4. **변환 스크립트의 경로 조건 누락** — PDF 가 조용히 fpdf2 폴백으로 떨어져 품질만 나빠집니다. 검증 8번의 문구로 잡습니다.
5. **`git worktree repair` 를 믿는 것** — 반대 방향으로 고칩니다(A-2 4단계).
6. **레인 상한에 걸려 worktree 생성이 막히는 것** — 이전과 무관한 기존 문제지만 이전 직후에 드러납니다. `merged` 로 멈춘 카드를 `wt-rm.sh` 로 `archived` 처리합니다.
7. **원본을 먼저 지우는 것** — 6단계 전부 통과 전에는 지우지 않습니다.

부수 현상: 옛 폴더 이름 변경이 "액세스 거부" 되는 경우가 있습니다(탐색기·문서 프로그램이 핸들을 잡고 있음). 동작에는 지장이 없습니다. 또 WSL 쪽 9p 캐시가 꼬여 `.old` 폴더가 `d?????????` 로 보일 수 있는데, 새 터미널을 열면 해소됩니다.

* * *

## 부록 B. 윈도우와의 접점 상세

### B-1. 변환 파이프라인

| 산출물 | 경로 | 폴백 순서 |
| --- | --- | --- |
| docx | pandoc(reference.docx) | — |
| pdf | ① libreoffice → ② **윈도우 Chrome/Edge headless(UNC)** → ③ fpdf2(맑은고딕) | ②가 정상 경로입니다 |
| pptx | ① marp(윈도우 Chrome) → ② **marp(WSL 리눅스 Chrome)** → ③ python-pptx | 방화벽 환경에서는 ②가 정상 |
| xlsx | openpyxl | — |
| hwp(입력) | pyhwp 추출 시도, 실패 가능 | **hwp 는 만들 수 없습니다.** docx 로 만들고 사용자가 한컴에서 저장합니다 |

`wslpath -w` 가 `/home/<사용자>/…` 를 `\\wsl.localhost\<배포판>\home\<사용자>\…` 로 바꿔 윈도우 프로그램에 넘깁니다. 리눅스 Chrome 을 쓸 때 공유 라이브러리가 모자라면 필요한 `.so` 를 한곳(예: `.fleet/chrome-libs/`)에 모아 `LD_LIBRARY_PATH` 로 지정하는 방법이 있습니다.

### B-2. 산출물함 규칙

- 경로: `OUTBOX/<프로젝트 구분>/<ID>/` — 구분은 `OFFICE_ID_PREFIXES` 를 따릅니다.
- **사본입니다.** 여기서 파일을 고쳐도 저장소에 반영되지 않습니다. 이 문장을 `office.md` 역할 파일과 발행 스크립트 출력에 모두 넣습니다.
- 복사 실패는 경고로만 처리하고 발행을 되돌리지 않습니다.

### B-3. 볼트

`VAULT` 는 **읽기 전용**입니다. `VAULT_GIT=0` 으로 두고, 일일 보고 스크립트도 `--vault` 를 명시할 때만 볼트에 절을 덧붙이게 합니다. 기본 동작이 볼트를 건드리는 일은 없어야 합니다.

* * *

* * *

## 부록 C. fe worktree 의 `node_modules` — 세 가지 방식과 Turbopack 함정

결론: **ext4 에서는 worktree 마다 `npm ci` 로 실물을 두는 것이 정석입니다.** 통합본을 심볼릭 링크로 재사용하면 설치를 아끼지만, Next 16 Turbopack 이 **workspace root 밖의 링크를 거부**해 `next build` 가 깨집니다. 링크를 꼭 쓰려면 `turbopack.root` 를 명시해야 합니다.

### C-1. 방식 비교

| 방식 | 설치 비용 | `tsc`·`lint` | `next build`·`next dev` | 언제 |
| --- | --- | --- | --- | --- |
| **worktree 마다 `npm ci`** | worktree 당 1회(ext4 에서는 감당 가능) | 통과 | 통과 | **기본값으로 권장** |
| 통합본 `node_modules` 심볼릭 링크 | 없음 | 통과 | **workspace root 를 벗어나면 실패** | root 를 맞출 수 있을 때만 |
| 메인 저장소 `node_modules` 복사 | 사본 크기만큼 디스크 | 통과 | 통과(네이티브 보충 필요) | `npm ci` 가 lock 불일치로 거부될 때 |

`package-lock.json` 이 `package.json` 과 어긋나면 `npm ci` 가 거부합니다. **락 파일 수정은 에이전트 금지 항목**이므로, 그때는 메인 작업본에서 사람이 `npm install` 로 락을 맞추거나 메인의 `node_modules` 를 복사해 씁니다.

### C-2. Turbopack workspace root 함정 (실측)

Turbopack 은 lockfile 이 있는 가장 가까운 상위 디렉터리를 workspace root 로 잡습니다. worktree 를 `<관제>/wt/<ID>/` 에 두고 `node_modules` 를 `<관제>/wt/integration/frontend/node_modules` 로 링크하면, root 가 `wt/<ID>/frontend` 라서 링크가 root 밖을 가리킵니다.

```
Symlink node_modules is invalid, it points out of the filesystem root
  at ... TurbopackInternalError
```

**두 가지 해법이 있습니다.**

| 해법 | 방법 | 평가 |
| --- | --- | --- |
| (a) `turbopack.root` 명시 | 프로젝트 `next.config` 에 `turbopack: { root: '<관제>/wt' }` | **정석.** 의도가 설정 파일에 드러납니다. 단 코드 저장소 파일을 고치는 것이라 카드·리뷰를 거칩니다 |
| (b) `wt/` 에 빈 lockfile | `<관제>/wt/package-lock.json` 에 `{}` | 저장소 파일을 건드리지 않고 root 를 `wt/` 로 끌어올립니다. **대신 의도가 어디에도 드러나지 않아 "잔재" 로 오인해 지우면 즉시 빌드가 깨집니다**(실제로 그렇게 깨뜨린 뒤 복원해 확인했습니다). 쓸 거라면 `wt-new.sh` 주석과 README 에 반드시 이유를 남기십시오 |

Next 는 이 상태에서 "multiple lockfiles" 경고를 냅니다. 빌드는 통과하지만 경고가 매번 나오므로, 여유가 생기면 (a) 로 옮기는 것이 맞습니다.

### C-3. 윈도우에서 설치된 `node_modules` 를 가져온 경우

윈도우에서 `npm install` 한 트리에는 리눅스 네이티브 패키지(`lightningcss` · `@tailwindcss/oxide` · `@next/swc` 의 `-linux-x64-gnu` 빌드)가 없어 WSL 빌드가 실패합니다. `package.json` · `package-lock.json` 을 건드리지 않고 해당 패키지만 `npm pack` 으로 받아 풀어 넣는 스크립트(`fix-natives.sh`)를 두고, `wt-new.sh` 가 복사 경로를 탈 때 자동으로 부르게 합니다.

### C-4. 의존성이 바뀐 뒤

의존성 추가는 오케스트레이터가 **메인 작업본에서만** 합니다. 그 뒤 열려 있는 fe worktree 는 각각 `npm ci` 를 다시 돌려야 하며, 링크 방식을 쓰고 있다면 통합본의 `node_modules` 를 재설치하기 전에 **열려 있는 worktree 가 없는지 먼저 확인**하십시오(링크가 전부 그 트리를 가리키고 있습니다).

* * *

## 부록 D. 장기기억 DB — 왜 하는가

결론: **여러 에이전트가 하나의 DB 를 공용 장기기억으로 쓰면, 세션이 끝나거나 에이전트가 바뀌어도 맥락이 끊기지 않습니다.** 이 관제에서 가장 비싼 일은 "새 세션이 이전 결정을 다시 알아내는 것" 이고, 장기기억은 그 비용을 줄입니다.

| 이점 | 이 관제에서의 모습 |
| --- | --- |
| 맥락 유지 | 인계 에이전트(4.4)가 인계 문서를 처음부터 다 읽지 않고, 카드 관련 기억만 꺼내 바로 이어감 |
| 에이전트 간 공유 | be 세션이 확인한 사실(포트·환경 함정)을 fe·office 세션이 그대로 씀. 같은 조사를 반복하지 않음 |
| 토큰 절약 | 전체 문서 대신 상위 `MEMORY_TOPK` 건만 프롬프트에 들어감. `*-hard`(Fable) 호출 전에 이미 푼 문제인지 먼저 확인 |
| 추적 가능 | 어떤 역할이 무엇을 근거(`source`)로 남겼는지 DB 에 남음 |

| 주의 | 대응 |
| --- | --- |
| 오래되거나 틀린 기억이 반복 사용됨 | 정본은 파일 · `valid_until` · `supersedes` · fact 쓰기 권한 제한 |
| 비밀 값 유출 | 쓰기 전 패턴 검사 · 접속 문자열 비출력 · 127.0.0.1 바인드 |
| 메모리 부담 | 0-0 (1) 산정식에 DB 몫 포함 · 임베딩은 나중에 켬 |
| 임베딩 모델 교체 | 벡터를 별도 테이블(`memory_embeddings`)에 모델별로 두어 본문은 그대로 · 새 모델 벡터를 채운 뒤 옛 모델 행 삭제 |
| DB 장애가 작업을 멈춤 | DB 가 없으면 기억 없이 진행(2-3 규격 9번) |

원본 메모: [[멀티에이전트-DB연동]]

관련 문서: [[멀티에이전트-오케스트레이션_구축_배포용_v3(WSL-ext4)]] (2026-09-15, v4 의 바로 앞 판) · [[멀티에이전트-오케스트레이션_구축_배포용_v2(토큰절약버전)]] (2026-09-11, `/mnt/c` 기준) · [[멀티에이전트 오케스트레이션 구성 v2(토큰 절약 버전)]] · [[멀티에이전트 오케스트레이션 구성 v1(전체 Fable버전)]]
