# SpecFlow — Spec 기반 자동화 파이프라인 구축 계획

> **상태**: 계획 (구현 착수 전, 확인 대기)
> **작성일**: 2026-08-09
> **목표**: `docs/`에 spec 문서를 두면 → 매일 정해진 시간(또는 수동 트리거)에 신규 spec을 감지 → 백엔드 → 프론트 → QA 순으로 자동 구현·검증 → feature 브랜치 push + draft PR 생성
> **재사용성**: 이 프로젝트 전용이 아니라, 어떤 프로젝트에든 설치해서 쓸 수 있는 독립 도구로 만든다

---

## 0. 확정된 결정 사항

| 항목 | 결정 |
|------|------|
| 단계 범위 | **구현까지 자동 수행** (코드 작성 + 커밋). 기획은 §13에서 앞단에 추가 |
| 실행 엔진 | **Antigravity headless (`agy -p`)** 주력, 스테이지·타깃 단위로 `claude`/`gemini`로 교체 가능 (§12). SDK 마이그레이션은 §10에 별도 계획 |
| 기획 자동화 | **기존 spec을 읽고 개선안 spec 초안을 생성.** 기본은 `status: draft`로 멈추고, `--auto-approve`를 쓸 때만 같은 run에서 개발까지 (§13) |
| 산출물 | **feature 브랜치 push + draft PR** |
| 스케줄러 | **launchd** (매일) + **CLI 수동 트리거** |
| 실행 시각 | **매일 01:00 (KST)**. 00:55에 `pmset`으로 자동 기상 (§7.1) |
| QA 실패 시 | **자가수정 루프 (최대 N회)**, 초과 시 중단 + draft PR에 실패 리포트 |
| 원격 push 범위 | 브랜치 push + **draft** PR까지. 머지는 항상 사람이 |
| spec 포맷 | **YAML frontmatter + 고정 섹션 템플릿** |
| 엔진 배포 | **GitHub private repo** → 새 컴퓨터는 clone + `install.sh` 한 번. 프로젝트별 설정·실행이력은 **커밋하지 않되**, 참고용 예시 config는 `examples/`로 동봉 (§3.1~3.2) |

---

## 1. 현재 프로젝트 실측 결과 (설계 제약)

계획 수립 전 실제로 확인한 사실들. 이 제약들이 설계를 크게 좌우한다.

### 1.1 리포지토리 구조 — **멀티 레포**

```
public-project/              ← git repo 아님 (단순 상위 디렉토리)
├── docs/                    ← spec 문서를 둘 위치
├── public-server/           ← git repo. origin: bbbbb0413/public-nest-server.git
├── public-front/            ← git repo. origin: bbbbb0413/public-front.git
├── public-python-server/    ← 아직 git repo 아님. **git init + push 예정** (§11-1 결정)
└── public-infra/
```

**함의**:
- 하나의 spec이 여러 repo를 건드릴 수 있으므로, **브랜치/PR은 repo별로 따로** 만들어야 한다.
- 파이프라인은 "repo 하나 = 단위"라는 가정을 버리고, **타깃(target) 목록**을 다루는 구조여야 한다.
- `public-python-server`도 repo가 되면 **세 타깃이 모두 동일하게** `repo: true`로 취급된다. 파이프라인 입장에서 예외 케이스가 사라지므로 구현이 단순해진다.
- 다만 **파이프라인 착수 전에 git 초기화·push가 끝나 있어야 한다.** 아직 remote가 없는 상태로 파이프라인을 돌리면 DELIVER 단계에서 그 타깃만 실패한다. → 부록 A 참고.

### 1.2 각 타깃의 검증 커맨드 (실측)

| 타깃 | 경로 | 빌드 | 린트 | 테스트 |
|------|------|------|------|--------|
| NestJS 백엔드 | `public-server` | `pnpm build` | `pnpm lint` | `pnpm test:all` (identity/payment/chat/gateway/ai 순차) |
| NestJS E2E | `public-server` | — | — | `pnpm test:e2e:all` (인프라 기동 필요) |
| Python AI 서버 | `public-python-server` | — | `ruff check` + `mypy` (strict) | `pytest` (`integration` 마커는 Mongo/Redis 필요) |
| 프론트엔드 | `public-front` | `pnpm build` (`tsc -b && vite build`) | `pnpm lint` | `pnpm test` (vitest) |

**함의**:
- `test:all`은 5개 스위트를 순차 실행 → 느리다. spec이 건드린 앱만 골라 돌리는 **선택적 실행**이 필요.
- E2E와 `integration` 마커 테스트는 **Docker 인프라 기동이 전제**. 야간 무인 실행에서 이게 가장 깨지기 쉬운 지점 → §6.3.
- Python은 `mypy strict` — 자동 생성 코드가 여기서 자주 막힐 것으로 예상. 자가수정 루프의 주 소비처.

### 1.3 환경

- `claude` CLI v2.1.226 설치됨 (`/Applications/cmux.app/Contents/Resources/bin/claude`) → **headless 실행 가능**
- `agy` (Antigravity) CLI v1.0.7 설치됨 (`~/.local/bin/agy`) → headless 실행 가능. 호출 형태가 `claude`와 달라 §12에서 별도로 다룬다
- `gemini` CLI v0.41.2 설치됨 (`/opt/homebrew/bin/gemini`) → 세 번째 선택지
- macOS (darwin 25.5.0), zsh
- 각 repo에 `CLAUDE.md`, `.claude/` 존재 → headless 실행 시 프로젝트 규약이 자동 적용됨 (큰 이점)

#### `agy` 실측 (2026-08-17)

빈 디렉토리에서 "파일 하나를 만들라"는 최소 과제로 확인한 결과다.

| 항목 | 결과 |
|---|---|
| 헤드리스 진입점 | `-p` / `--print`. 프롬프트를 **플래그 값**으로 받는다 |
| 프롬프트 전달 | stdin 파이프는 쓸 수 없다. `echo … \| agy -p --print-timeout 3m` 은 `--print-timeout` 을 프롬프트 문자열로 먹는다 |
| 플래그 순서 | 나머지 플래그가 `-p` **앞**에 와야 한다 |
| 무인 승인 | `--dangerously-skip-permissions` 가 없으면 도구 호출이 자동 거부되고 아무 일도 일어나지 않는다 |
| **작업 디렉토리** | `--add-dir` 가 **필수**다. 셸의 cwd 를 작업공간으로 잡지 않는다 |
| 종료 코드 | 성공·무동작 모두 **0** |
| 기본 타임아웃 | `--print-timeout` 5분 |
| 모델 | Gemini 3.6/3.5 Flash, 3.1 Pro, Claude Sonnet/Opus 4.6, GPT-OSS 120B (`agy models`) |

**설계를 좌우하는 두 가지.**

`--add-dir` 를 빼고 돌리면 "hello.txt 파일을 생성하고 내용으로 OK를 작성했습니다" 라고 답한 뒤 파일을 만들지 않는다. 디스크 어디에도 없었고 종료 코드는 0 이었다. 엔진이 종료 코드만 보면 빈 스테이지를 성공으로 기록하고 다음 단계로 넘어간다. → §12.3 무동작 감지가 필요한 이유다.

프롬프트를 인자로 넘겨야 하므로 `_invoke_agent` 의 stdin 전달을 그대로 쓸 수 없다. 에이전트마다 전달 방식이 다르다는 것을 설정으로 표현해야 한다. → §12.1

---

## 2. 전체 아키텍처

```
                    ┌──────────────────────────────┐
   매일 01:00 ─────▶│  launchd (com.specflow.daily) │
   수동 트리거 ─────▶│  또는  $ specflow run         │
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │  0. PLAN  (선택, §13)        │
                    │  기존 spec + 커밋 이력을 읽고  │
                    │  개선안 spec 초안 1건 작성     │
                    │  → status: draft 로 멈춤      │
                    │  (--auto-approve 면 ready)   │
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │  1. DETECT                   │
                    │  specs 디렉토리 스캔          │
                    │  state ledger와 대조          │
                    │  → 신규/변경된 spec 목록      │
                    └──────────────┬───────────────┘
                                   ▼ (spec 하나씩 순차)
                    ┌──────────────────────────────┐
                    │  2. PLAN                     │
                    │  spec 파싱 → 실행 계획 생성   │
                    │  타깃별 브랜치 생성           │
                    └──────────────┬───────────────┘
                                   ▼
     ┌─────────────────────────────────────────────────────┐
     │  3. STAGES (순차)                                    │
     │                                                      │
     │   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
     │   │ backend  │──▶│ frontend │──▶│    qa    │        │
     │   └────┬─────┘   └────┬─────┘   └────┬─────┘        │
     │        │              │              │              │
     │      gate           gate           gate             │
     │   (build/lint/    (build/lint/   (전체 테스트+       │
     │    unit test)      unit test)     E2E)              │
     │        │              │              │              │
     │        └──────────────┴──────────────┘              │
     │                       │ 실패                         │
     │                       ▼                              │
     │              ┌─────────────────┐                     │
     │              │  REPAIR LOOP    │ 최대 N회            │
     │              │  실패로그 → 수정 │                     │
     │              └─────────────────┘                     │
     └─────────────────────────┬───────────────────────────┘
                               ▼
                    ┌──────────────────────────────┐
                    │  4. DELIVER                  │
                    │  타깃별 push + draft PR       │
                    │  실행 리포트 첨부             │
                    │  state ledger 갱신            │
                    └──────────────────────────────┘
```

각 스테이지는 **`claude -p` 한 번의 headless 호출**이며, 스테이지 사이 상태는 **파일(run 디렉토리)** 로 전달한다.

---

## 3. 설치 구조 — 재사용 가능하게

### 3.1 두 개의 층

**① 엔진 (전역, 프로젝트 무관)** — `~/.specflow/` = **private repo의 clone**

```
~/.specflow/                      ← git clone <private repo>
├── install.sh                    # 새 컴퓨터 부트스트랩 (§3.3)
├── bin/
│   └── specflow                  # CLI 진입점 (bash)
├── lib/
│   ├── detect.sh                 # spec 스캔 + ledger 대조
│   ├── orchestrate.sh            # 스테이지 실행 루프
│   ├── stage.sh                  # claude -p 단일 스테이지 실행
│   ├── gate.sh                   # 검증 커맨드 실행 + 결과 파싱
│   ├── repair.sh                 # 자가수정 루프
│   ├── deliver.sh                # git 브랜치/커밋/push/PR
│   └── report.sh                 # 리포트 생성
├── prompts/
│   ├── backend.md                # 스테이지별 기본 프롬프트 템플릿
│   ├── frontend.md
│   ├── qa.md
│   └── repair.md
├── templates/                    # init이 채워 넣을 빈 뼈대
│   ├── spec.template.md          # spec 문서 템플릿
│   ├── config.template.yaml      # 주석만 있는 설정 스켈레톤
│   └── com.specflow.daily.plist  # launchd 템플릿
├── examples/                     # 실제 동작하는 완성 config (베껴 쓰는 용도)
│   ├── multi-repo.yaml           # 멀티 레포 + Nest/Python/Vite  ← 이 프로젝트 형태
│   ├── single-repo-monorepo.yaml # 단일 repo 안에 apps/* 구조
│   └── minimal.yaml              # 타깃 1개, gate 2개짜리 최소 예시
├── detectors/                    # specflow init의 스택 자동 감지 규칙
│   ├── node.sh                   # package.json → 빌드/린트/테스트 커맨드 추출
│   ├── python.sh                 # pyproject.toml → ruff/mypy/pytest
│   └── ...
└── .gitignore                    # runs/ 제외
```

> **`templates/`와 `examples/`는 역할이 다르다.**
> `templates/config.template.yaml`은 주석만 있는 빈 뼈대로, `specflow init`이 감지 결과를 채워 넣는 대상이다. `examples/*.yaml`은 **그대로 복사해서 경로만 바꾸면 도는 완성본**이다.
> 프로젝트별 config를 커밋하지 않기로 했으므로, 새 컴퓨터에서 자동 감지가 애매할 때 참고할 실물이 하나도 없으면 매번 처음부터 고민하게 된다. 특히 **멀티 레포는 자동 감지가 가장 헷갈리는 형태**라 `multi-repo.yaml`이 사실상 필수다.
> 단, 예시에는 **실제 사용자 이름·머신 경로·사내 repo 주소를 넣지 않는다.** `<owner>`, `~/work/<project>` 같은 자리표시자로 일반화한다 (private repo라도 이식성과 위생 문제).

**② 프로젝트 설정 (로컬 전용, 커밋 안 함)** — `<project>/.specflow/`

```
<project>/.specflow/
├── config.yaml                   # 이 프로젝트의 타깃/커맨드/규칙   ← 커밋 안 함
├── state.json                    # 처리된 spec ledger               ← 커밋 안 함
├── run.lock                      # 동시 실행 방지
└── runs/                         # 실행 로그·리포트                 ← 커밋 안 함
```

> **왜 프로젝트 설정은 커밋하지 않나**
> private repo의 목적은 "새 컴퓨터에서 빠르게 복원"이다. `config.yaml`은 그 프로젝트의 경로·커맨드일 뿐이고 `specflow init`이 다시 만들어 줄 수 있으며, `state.json`·`runs/`는 그 맥의 실행 이력이라 다른 컴퓨터로 옮길 의미가 없다. 엔진만 담으면 repo가 가볍고 커밋 노이즈도 없다.
>
> **대신 `specflow init`의 자동 감지 품질이 중요해진다.** 새 맥에서 config를 손으로 다시 쓰는 게 아니라 자동 생성 + 몇 줄 수정으로 끝나야 한다. `detectors/`가 그 역할을 맡는다 (§3.4).

> **재사용 원리**: 엔진은 프로젝트를 전혀 모른다. "무엇을 어떤 커맨드로 검증할지"는 전부 `config.yaml`이 알려준다.

### 3.2 새 컴퓨터에 설치하기

이것이 private repo를 두는 목적이다. 목표는 **명령 두 줄**.

```bash
git clone git@github.com:<owner>/specflow.git ~/.specflow
~/.specflow/install.sh
```

`install.sh`가 하는 일:

1. **의존성 점검** — `claude`, `git`, `gh`, `node`/`pnpm`, `python`, `docker` 존재 확인. 없는 것은 설치 명령을 안내하고 **중단하지 않는다** (해당 타깃을 쓰지 않을 수도 있으므로 경고만).
2. **PATH 등록** — `~/.zshrc`에 `export PATH="$HOME/.specflow/bin:$PATH"` 추가 (이미 있으면 skip).
3. **launchd 등록 여부 확인** — 대화형으로 물어보고, 원하면 plist를 생성·로드. 프로젝트 경로는 이때 입력받는다.
4. **`gh` 인증 상태 확인** — 미인증이면 `gh auth login` 안내 (대화형이라 자동화 불가).
5. 완료 후 다음 단계 안내: `cd <project> && specflow init`

새 프로젝트를 파이프라인에 붙일 때:

```bash
cd ~/work/some-project
specflow init          # 스택 감지 → .specflow/config.yaml 생성
$EDITOR .specflow/config.yaml   # 타깃·커맨드 확인/수정
specflow run --dry-run          # 감지 결과 확인
```

**이식성을 위해 지켜야 할 것**:
- 엔진 스크립트에 **절대 경로를 하드코딩하지 않는다.** `$HOME`, `$SPECFLOW_HOME`, config의 상대 경로만 사용.
- 사용자 이름·특정 맥의 경로가 repo에 들어가지 않게 한다 (plist는 템플릿으로 두고 `install.sh`가 치환).
- private repo라도 **토큰·비밀정보는 절대 넣지 않는다.** 인증은 `gh`의 키체인, API 키는 각 프로젝트 `.env`에 맡긴다.

### 3.3 `specflow init` — 스택 자동 감지

config를 커밋하지 않는 대신 여기에 투자한다. 프로젝트 루트를 훑어 타깃 후보를 만든다.

| 발견한 것 | 추론하는 타깃 |
|---|---|
| `package.json` + `nest-cli.json` | NestJS 백엔드. scripts에서 `build`/`lint`/`test*` 추출 |
| `package.json` + `vite.config.*` | 프론트엔드. `build`/`lint`/`test` |
| `pyproject.toml` | Python. `[tool.ruff]`/`[tool.mypy]`/`[tool.pytest.ini_options]` 유무로 gate 구성 |
| `go.mod` / `Cargo.toml` | Go / Rust |
| `docker-compose.yml` | `infra.up`/`down` 후보 |
| 하위 디렉토리마다 `.git` | **멀티 레포**로 판정 → 타깃별 `repo: true` + remote 추출 |

감지 결과는 **주석이 달린 config.yaml 초안**으로 출력하고, 확신이 낮은 항목은 `# TODO: 확인 필요` 를 붙인다. 자동 감지를 신뢰하지 말고 반드시 사람이 한 번 훑도록 유도한다.

### 3.4 CLI 인터페이스

```bash
specflow init                 # 현재 디렉토리에 .specflow/ 스캐폴딩 (스택 자동 감지 시도)
specflow doctor               # 실행 전 점검 — 게이트가 지금 통과하는지 기준선 측정
specflow plan                 # 기획: 기존 spec을 읽고 개선안 spec 초안 1건 생성 (§13)
specflow plan --auto-approve  # 초안을 ready로 승격하고 이어서 개발까지
specflow run                  # 신규 spec 감지 → 전체 파이프라인 실행 (launchd가 호출하는 것)
specflow run <spec-file>      # 특정 spec만 강제 실행 (감지 무시)
specflow run --plan           # 기획 → 개발을 한 번에
specflow run --dry-run        # 감지 결과와 실행 계획만 출력, 코드는 안 건드림
specflow status               # 처리 이력, 진행 중/실패한 run 목록
specflow retry <spec-id>      # 지난번 실패·중단을 재개. id 생략 시 전부
specflow schedule install     # launchd plist 설치
specflow schedule uninstall
```

> `retry`의 인자는 run-id가 아니라 **spec-id**다. 재개 단위가 run이 아니라 spec이기 때문이다. 같은 spec을 다시 돌리면 새 run 디렉토리가 생기고 이전 run의 로그는 그대로 남는다.

---

## 4. spec 문서 계약

### 4.1 위치

`<project>/docs/specs/*.md` (config에서 변경 가능)

기존 `docs/` 루트에 있는 다른 문서(`user-prompt-settings-plan.md` 등)와 섞이지 않도록 **전용 하위 디렉토리**를 쓴다.

### 4.2 템플릿

```markdown
---
id: SPEC-001                       # 필수. 고유 ID. 브랜치명에 사용
title: 사용자 프롬프트 설정 기능     # 필수
status: ready                      # ready | draft | done | blocked
                                   #   ready인 것만 파이프라인이 집는다
targets:                           # 필수. 어떤 타깃을 건드리는지
  - backend-nest
  - frontend
stages: [backend, frontend, qa]    # 선택. 생략 시 config 기본값
priority: normal                   # 선택. high면 같은 날 먼저 처리
max_repair_attempts: 3             # 선택. 생략 시 config 기본값
---

## 배경 / 문제

왜 필요한가. 현재 무엇이 불편한가.

## 요구사항

- [ ] 기능적 요구사항을 체크박스로. 이게 QA 스테이지의 1차 검증 목록이 된다.
- [ ] ...

## 비요구사항 (Out of scope)

이번에 하지 않을 것. 에이전트의 과잉 구현을 막는 가장 중요한 섹션.

## 백엔드

API 스펙, 데이터 모델, 이벤트, 영향받는 모듈 경로.

## 프론트엔드

화면/컴포넌트, 상태 관리, 호출할 API.

## 수용 기준 (Acceptance Criteria)

- Given ... When ... Then ...
- QA 스테이지가 테스트로 변환하는 대상.

## 참고

관련 문서·이슈 링크.
```

### 4.3 검증

`specflow run`은 spec을 실행에 넘기기 전에 **frontmatter 스키마를 검증**한다. `id`/`title`/`targets` 누락, 알 수 없는 target 이름 → 그 spec은 스킵하고 리포트에 사유를 남긴다. (잘못된 spec으로 야간에 엉뚱한 코드가 생기는 것 방지)

---

## 5. 프로젝트 설정 (`config.yaml`)

이 프로젝트에 실제로 적용할 초안:

```yaml
version: 1

specs:
  dir: docs/specs
  glob: "*.md"

git:
  branch_pattern: "feat/{spec_id}-{slug}"
  commit_pattern: "{type}: {title} ({spec_id})"
  base_branch: main            # 타깃별 override 가능
  push: true
  pr: draft                    # draft | ready | none

targets:
  backend-nest:
    path: public-server
    repo: true                 # 독립 git repo
    remote: origin
    gates:
      lint:  "pnpm lint"
      build: "pnpm build"
      test:  "pnpm test:all"
    # 선택적 실행: spec이 건드린 앱만
    scoped_test:
      apps/identity:     "pnpm test:identity"
      apps/payment:      "pnpm test:payment"
      apps/chat-service: "pnpm test:chat"
      apps/gateway:      "pnpm test:gateway"
      apps/ai-service:   "pnpm test:ai"

  backend-python:
    path: public-python-server
    repo: true                 # git init + push 예정 (§11-1). 착수 전 완료 필요
    remote: origin
    gates:
      lint: "ruff check src tests"
      type: "mypy src"
      test: "pytest -m 'not integration'"

  frontend:
    path: public-front
    repo: true
    remote: origin
    gates:
      lint:  "pnpm lint"
      build: "pnpm build"
      test:  "pnpm test"

stages:
  - name: backend
    targets: [backend-nest, backend-python]
    prompt: backend            # ~/.specflow/prompts/backend.md
  - name: frontend
    targets: [frontend]
    prompt: frontend
  - name: qa
    targets: [backend-nest, backend-python, frontend]
    prompt: qa
    # 1단계는 유닛 테스트만 (§11-3 결정). 안정화 후 아래 두 줄의 주석을 해제한다.
    # extra_gates:
    #   backend-nest: "pnpm test:e2e:all"
    requires_infra: false      # true로 바꾸면 infra.up/down이 붙는다

infra:
  up:     "docker compose up -d"
  down:   "docker compose down"
  health: "docker compose ps --format json"
  wait_timeout: 180

repair:
  max_attempts: 3
  # 실패 로그에서 에이전트에게 넘길 최대 줄 수 (컨텍스트 폭발 방지)
  log_tail_lines: 200

limits:
  stage_timeout_minutes: 30
  run_timeout_minutes: 120
  max_specs_per_run: 3         # 하룻밤에 3개까지만

notify:
  macos_notification: true
  log_file: .specflow/runs/latest.log
```

---

## 6. 실행 흐름 상세

### 6.1 DETECT — 무엇을 "신규"로 볼 것인가

`state.json` ledger (필드 구조, 값은 예시):

```json
{
  "specs": {
    "SPEC-001": {
      "file": "docs/specs/001-user-prompt-settings.md",
      "content_hash": "sha256:abc123...",
      "status": "delivered",
      "last_run": "2026-08-09T09:00:00+09:00",
      "run_id": "20260809-090000-SPEC-001",
      "prs": {
        "backend-nest": "https://github.com/<owner>/<repo>/pull/42",
        "frontend": "https://github.com/<owner>/<repo>/pull/17"
      }
    }
  }
}
```

- `last_run`: ISO 8601, **오프셋 포함** (`+09:00`). 로컬 시간 혼동 방지.
- `run_id`: `YYYYMMDD-HHMMSS-<SPEC-ID>` — 정렬 가능하고 run 디렉토리명과 동일.
- `status`: `delivered` | `failed` | `skipped` | `running`

**신규 처리 대상 판정**:
1. frontmatter `status: ready` 인가? (아니면 스킵)
2. ledger에 `id`가 없다 → **신규**
3. ledger에 있지만 `content_hash`가 다르다 → **변경됨**. 기본은 스킵하고 리포트에만 남긴다. (이미 PR이 열려 있는데 덮어쓰면 위험) `--allow-modified` 플래그로만 재처리.
4. ledger 상태가 `failed`다 → 스킵. `specflow retry`로만 재개.

> 파일명·mtime이 아니라 **frontmatter id + 내용 해시** 기준. 파일을 옮기거나 이름을 바꿔도 중복 실행되지 않는다.

### 6.2 STAGE — headless 호출의 실제 형태

각 스테이지는 이런 식으로 실행된다:

```bash
claude -p "$(render_prompt backend.md)" \
  --output-format stream-json \
  --permission-mode acceptEdits \
  --allowedTools "Read,Write,Edit,Glob,Grep,Bash(pnpm *),Bash(git *)" \
  --add-dir "$PROJECT_ROOT/docs/specs" \
  > "$RUN_DIR/backend.stream.json"
```

- **작업 디렉토리는 타깃 경로** (`public-server`) → 그 repo의 `CLAUDE.md`가 자동 적용됨
- `--allowedTools`로 도구를 화이트리스트 → 야간 무인 실행의 안전장치
- 스테이지 프롬프트에는 항상 다음이 포함된다:
  - spec 문서 전문
  - 이전 스테이지의 요약 (`$RUN_DIR/backend.summary.md`)
  - "완료 시 `$RUN_DIR/<stage>.summary.md`에 변경 파일 목록과 결정 사항을 남길 것"
  - "커밋은 하되 push는 하지 말 것" (push/PR은 DELIVER가 전담)

**스테이지 간 상태 전달**은 컨텍스트 승계가 아니라 **파일 기반**이다. 각 스테이지는 깨끗한 컨텍스트로 시작하고, 앞 단계의 결과는 요약 문서로만 받는다. (컨텍스트 폭발 방지 + 재시도 가능)

```
runs/20260809-090000-SPEC-001/
├── spec.md                  # 스냅샷 (원본이 바뀌어도 재현 가능)
├── plan.json                # 타깃·브랜치·스테이지 계획
├── backend.summary.md
├── backend.stream.json
├── backend.gate.json        # {lint: pass, build: pass, test: fail}
├── frontend.summary.md
├── qa.summary.md
├── repair-1.summary.md
└── report.md                # PR 본문이 되는 최종 리포트
```

### 6.3 GATE — 검증

각 스테이지 종료 직후 config의 `gates`를 순서대로 실행하고 결과를 `<stage>.gate.json`에 기록.

- gate는 **엔진이 직접 실행**한다 (에이전트에게 "테스트 돌려봐"라고 맡기지 않음). 에이전트가 "테스트 통과했습니다"라고 착각하는 문제를 원천 차단.
- `requires_infra: true` 스테이지 진입 전 `infra.up` 실행 → health check 폴링 → 타임아웃 시 **해당 gate를 skip 처리하고 리포트에 명시** (실패로 처리하면 인프라 문제로 매일 밤 파이프라인이 죽는다)
- run 종료 시 `infra.down`

### 6.4 REPAIR — 자가수정 루프

```
gate 실패
  ↓
실패 로그 tail N줄 + 실패한 커맨드 + 해당 스테이지 요약을 프롬프트로 구성
  ↓
claude -p (repair.md 프롬프트, 작업 디렉토리 = 실패한 타깃)
  ↓
gate 재실행
  ↓
통과 → 다음 스테이지 / 실패 → attempt++ 후 반복 (최대 max_attempts)
  ↓
초과 시: 루프 중단, 현재 상태 그대로 DELIVER로 진행 (draft PR + 실패 리포트)
```

**루프 폭주 방지**:
- 각 시도의 gate 실패 지문(실패 테스트 이름 집합)을 해시로 저장. **동일 지문이 2회 연속** 나오면 진전 없음으로 판단하고 max_attempts 전에 조기 중단.
- repair 프롬프트에는 "테스트를 삭제하거나 skip 처리해서 통과시키지 말 것"을 명시적 금지 조항으로 넣는다. (에이전트가 가장 흔히 취하는 편법)
- repair 시도 전후로 `git diff --stat`을 기록해 어떤 수정이 있었는지 리포트에 남긴다.

### 6.5 DELIVER — 브랜치 / PR

멀티 레포이므로 **타깃별로 독립 수행**:

```
for target in 변경된 타깃들:
    cd $target.path
    git push -u origin feat/SPEC-001-user-prompt-settings
    gh pr create --draft \
        --title "feat: 사용자 프롬프트 설정 기능 (SPEC-001)" \
        --body-file $RUN_DIR/report.md \
        --base main
```

- **변경이 없는 타깃은 브랜치도 PR도 만들지 않는다**
- 여러 repo에 PR이 생기면 각 PR 본문 하단에 **상호 링크**를 넣는다 ("관련 PR: frontend#17")
- `gh` CLI 미설치·미인증 시 push까지만 하고 리포트에 PR 생성 실패를 명시

### 6.6 PR 본문 (`report.md`) 구성

```markdown
## SPEC-001 사용자 프롬프트 설정 기능

🤖 SpecFlow 자동 생성 · run `20260809-090000-SPEC-001`
⚠️ 자동 생성된 코드입니다. 머지 전 반드시 리뷰하세요.

### 요구사항 대비 구현 현황
- [x] 사용자별 프롬프트 저장
- [x] 프롬프트 조회 API
- [ ] ⚠️ 프롬프트 버전 관리 — 미구현 (사유: spec 모호, 판단 필요)

### 스테이지 결과
| 스테이지 | 결과 | lint | build | test |
|---------|------|------|-------|------|
| backend  | ✅ | ✅ | ✅ | ✅ |
| frontend | ✅ | ✅ | ✅ | ✅ |
| qa       | ⚠️ | — | — | E2E 2건 실패 (repair 3회 초과) |

### 변경 파일
...

### ⚠️ 사람이 확인해야 할 것
- E2E `payment.e2e-spec.ts:44` 실패 — 결제 목 데이터 문제로 추정
- 인프라 기동 타임아웃으로 integration 테스트 skip됨

### 실행 로그
`.specflow/runs/20260809-090000-SPEC-001/`
```

---

## 7. 스케줄링

### 7.1 launchd

`~/Library/LaunchAgents/com.specflow.daily.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key><string>com.specflow.daily</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/seungwonkang/.specflow/bin/specflow</string>
        <string>run</string>
        <string>--project</string>
        <string>/Users/seungwonkang/Documents/work/public-project</string>
    </array>
    <!-- 시스템 로컬 타임존 기준. 이 맥은 Asia/Tokyo(UTC+9) = KST와 동일 오프셋 -->
    <key>StartCalendarInterval</key>
    <dict><key>Hour</key><integer>1</integer><key>Minute</key><integer>0</integer></dict>
    <key>StandardOutPath</key><string>/Users/seungwonkang/.specflow/runs/daily.out.log</string>
    <key>StandardErrorPath</key><string>/Users/seungwonkang/.specflow/runs/daily.err.log</string>
    <key>EnvironmentVariables</key>
    <dict><key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string></dict>
</dict>
</plist>
```

**launchd 함정과 대응**:
- launchd는 **로그인 셸 환경을 상속하지 않는다** → `PATH`에 `claude`, `pnpm`, `node`, `gh`, `docker` 경로를 명시. 래퍼 스크립트 첫 줄에서 필수 커맨드 존재 여부를 전부 확인하고, 없으면 즉시 실패 + 알림.
- **⚠️ 타임존**: `StartCalendarInterval`은 UTC가 아니라 **시스템 로컬 타임존** 기준이다. 이 맥은 현재 `Asia/Tokyo`(JST)로 설정돼 있는데, JST와 KST는 **둘 다 UTC+9이고 서머타임이 없어** 오프셋이 완전히 동일하다. 따라서 `Hour: 1`은 KST 01:00과 일치한다. 다만 나중에 시스템 타임존을 바꾸면 실행 시각이 따라 움직이므로, 래퍼 스크립트 시작 시 현재 타임존을 로그에 기록하고 UTC+9가 아니면 경고를 남긴다.
- 맥이 잠들어 있으면 `StartCalendarInterval`은 **깨어난 직후 1회 실행**된다 (cron과 달리 스킵되지 않음). 이게 launchd를 택한 이유.
- **✅ 자동 기상 예약**: 01:00은 맥이 잠들어 있을 시각이므로 5분 먼저 깨운다. 맥이 상시 전원 연결 상태이므로 이 방식을 택한다.

  ```bash
  sudo pmset repeat wake MTWRFSU 00:55:00
  ```

  실측으로 확인한 전제 (2026-08-09 기준):
  - 기존 `pmset repeat` 예약 **없음** → 덮어쓸 사용자 예약이 없다. (`pmset -g sched`에 보이던 항목들은 캘린더·분석 앱이 잡은 **일회성** wake라 `repeat`과 충돌하지 않는다)
  - AC 전원 연결됨, AC 설정에서 `powernap 1` / `womp 1` / `hibernatemode 3` → 예약 wake가 정상 동작하는 조건.

  주의:
  - `pmset repeat`은 **반복 일정을 하나만** 보유한다. 나중에 다른 반복 예약을 걸면 이게 사라진다. `install.sh`는 설정 전 `pmset -g sched`를 확인해 기존 repeat이 있으면 **덮어쓰지 않고 경고**한다.
  - **잠자기(sleep)에서만 깨운다.** 완전히 종료된 상태는 깨우지 못한다.
  - 깨어난 뒤 화면은 꺼진 채(dark wake)일 수 있으나 백그라운드 실행에는 지장 없다. 깨운 뒤 자동으로 다시 자지는 않고, 유휴 시간이 지나면 알아서 잠든다.
  - 배터리 전원으로 바뀌면 예약이 무시될 수 있다. 파이프라인 시작 시 `pmset -g batt`로 전원 상태를 로그에 남긴다.
- **동시 실행 방지**: `.specflow/run.lock` 파일 락. 이전 run이 살아 있으면 새 run은 즉시 종료.
- 야간 실행이므로 완료 알림은 놓치기 쉽다. `specflow status`가 **아침에 결과를 확인하는 주 경로**가 되도록 출력을 설계한다.

### 7.2 수동 트리거

```bash
specflow run                              # 스케줄과 동일하게
specflow run docs/specs/001-xxx.md        # 특정 spec 강제
specflow run --dry-run                    # 뭐가 잡히는지 확인만
```

여기에 더해 Claude Code 세션 안에서 `/specflow` 슬래시 커맨드로도 부를 수 있게 얇은 래퍼를 둔다 (`.claude/commands/specflow.md`).

---

## 8. 안전장치

야간 무인 실행이라 이 부분이 가장 중요하다.

| 위험 | 대응 |
|------|------|
| 엉뚱한 브랜치에 커밋 | 스테이지 시작 전 반드시 `base_branch`에서 새 브랜치 생성. 현재 브랜치가 dirty면 **run 자체를 중단** |
| main에 push | `deliver.sh`에서 브랜치명이 `branch_pattern`과 일치하지 않으면 push 거부 (하드 가드) |
| 무한 루프 / 토큰 폭주 | 스테이지 타임아웃 30분, run 타임아웃 120분, run당 spec 최대 3개, repair 최대 3회 + 조기 중단 |
| 테스트를 지워서 통과시킴 | repair 프롬프트 금지 조항 + `git diff`에서 테스트 파일 **삭제** 감지 시 해당 repair를 롤백하고 실패 처리 |
| 위험한 명령 실행 | `--allowedTools` 화이트리스트. `Bash(rm *)`, `Bash(git push *)` 등은 미허용 |
| 비밀정보 커밋 | DELIVER 직전 `git diff`에 대해 secret 스캔 (`.env`, 키 패턴). 검출 시 push 중단 |
| 인프라 미기동으로 매일 실패 | health check 타임아웃 시 해당 gate만 skip + 리포트 명시 (run은 계속) |
| 킬 스위치 | `.specflow/DISABLED` 파일이 존재하면 예외 없이 즉시 종료 |
| 실행 결과를 모름 | 완료·실패 시 macOS 알림 + `specflow status`로 언제든 확인 |

**첫 2주는 `pr: none` (푸시 안 함, 로컬 브랜치만)** 으로 운영하며 결과 품질을 관찰한 뒤 draft PR을 켜는 것을 권장한다.

---

## 9. 구현 단계 (제안 순서)

| Phase | 내용 | 산출물 | 예상 |
|-------|------|--------|------|
| ~~**P0**~~ ✅ | spec 템플릿·frontmatter 스키마 확정, 검증기 | `SCHEMA.md`, `templates/spec.template.md`, `lib/validate-spec.sh`(+`_validate_spec.py`), `tests/` 39개 통과 | 완료 |
| ~~**P1**~~ ✅ | private repo 생성 + 엔진 뼈대 + CLI + config 로더 + DETECT + `--dry-run` | `bin/specflow`, `lib/_yaml.py`, `_load_config.py`, `_state.py`, `_detect.py`, 테스트 195개 통과 | 완료 |
| ~~**P2**~~ ✅ | 스테이지 실행 (`claude -p` + gate + 커밋). 실제 claude 로 검증 완료 | `_gate.py`, `_run.py`, `_prompt.py`, `_stage.py`, `_orchestrate.py`, `prompts/*`, 테스트 294개 | 완료 |
| ~~**P3**~~ ✅ | 멀티 타깃 오케스트레이션 — 아티팩트 덮어쓰기 버그 수정, 진행 상태 기록 | `_run.py`(경로에 타깃 포함, `progress`), `_prompt.py`, 테스트 309개 | 완료 |
| ~~**P4**~~ ✅ | 자가수정 루프 + 조기 중단 + 편법 롤백 | `_fingerprint.py`, `_repair.py`, `prompts/repair.md`, 테스트 367개 | 완료 |
| ~~**P5**~~ ✅ | 멀티레포 push + draft PR + 리포트 + 비밀정보 스캔 | `_secrets.py`, `_report.py`, `_deliver.py`, 테스트 451개 | 완료 |
| **P6** ⏸ | launchd 설치·자동 기상·알림 | plist, `schedule` 서브커맨드 | **보류** — 당장 자동 실행할 단계가 아니라 건너뜀 (2026-08-12). 스케줄 없이 `specflow run` 수동 실행으로 충분. 켤 때 §7 설계 그대로 구현하면 된다 |
| ~~**P7**~~ ✅ | `install.sh` + `specflow init` (스택 자동 감지) — **재사용성·이식성 완성** | `_detect_stack.py`, `_cli_init.py`, `install.sh`, `examples/{multi-repo,minimal}.yaml`, 테스트 491개 | 완료. clone→install→init→dry-run 전 구간을 새 머신 시나리오로 검증 |
| ~~**P8**~~ ✅ | 실 spec 1건으로 E2E 리허설 (SPEC-001 원화 표시 통일) | 게이트 범위 축소, `specflow retry` 구현, 에이전트 자체 커밋 시 내보내기 누락 버그 수정, 테스트 513개 | 완료 (2026-08-17). 백엔드→프론트→QA 4개 스테이지가 자가수정 없이 통과 |
| **P9** | 에이전트 계층 — `agy` 주력 + 스테이지별 교체 (§12) | `_agent.py`, config `agent:` 절, doctor 확장, 무동작 감지 | 1일 |
| **P10** | 기획 스테이지 — spec 문서 자동 생성 (§13) | `prompts/plan.md`, `_cli_plan.py`, `--auto-approve` | 1~2일 |

**검증 방식**: 각 Phase는 `--dry-run` → 샌드박스 브랜치 실행 → 실제 실행 순으로 확인. P2까지는 반드시 사람이 지켜보는 앞에서 수동 실행만.

---

## 10. Claude Agent SDK 마이그레이션 계획 (2단계)

headless CLI로 먼저 동작시키되, 아래 신호가 보이면 SDK로 옮긴다.

### 10.1 마이그레이션 트리거

- 스테이지 간 상태를 파일로만 넘기는 게 한계에 부딪힐 때 (요약 손실로 품질 저하)
- 여러 spec을 **병렬** 처리하고 싶을 때
- `stream-json` 파싱으로 진행 상황을 추적하는 bash 코드가 감당이 안 될 때
- 토큰 사용량·비용을 스테이지 단위로 계측·제어하고 싶을 때
- 실패 지점 재개(resume) 로직이 복잡해질 때

### 10.2 마이그레이션이 쉽도록 지금 지켜야 할 설계 원칙

1. **엔진 스크립트를 얇게 유지한다.** 판단 로직은 전부 프롬프트(`prompts/*.md`)와 설정(`config.yaml`)에 둔다. SDK로 갈 때 이 둘은 **그대로 재사용**된다.
2. **스테이지 = 순수 함수**로 취급. `(spec, 이전 요약, 작업 디렉토리) → (변경, 요약, gate 결과)`. 이 시그니처를 유지하면 SDK의 `query()` 호출로 1:1 치환된다.
3. **런타임 산출물 포맷(`run` 디렉토리 구조, `state.json`)을 먼저 확정한다.** 엔진이 bash든 TS든 이 포맷만 지키면 `specflow status`, 리포트, 재시도는 그대로 동작한다.
4. 셸 전용 기능(파이프, 서브셸 트릭)에 로직을 의존시키지 않는다.

### 10.3 마이그레이션 시 얻는 것

| 항목 | headless CLI | Agent SDK |
|------|-------------|-----------|
| 스테이지 간 컨텍스트 | 파일 요약만 | 세션 유지 / 선택적 승계 |
| 병렬 처리 | 어려움 (프로세스 관리) | 자연스러움 |
| 진행 추적 | stream-json 파싱 | 메시지 스트림 직접 소비 |
| 재개 | run 디렉토리 기반 수동 | 세션 resume |
| 비용 계측 | 로그 후처리 | 응답 메타데이터 |
| 커스텀 도구 | 불가 | MCP/커스텀 툴 주입 가능 |

### 10.4 마이그레이션 실행 계획 (착수 시)

1. `lib/stage.sh` → `engine/stage.ts` 로 먼저 1개만 치환. CLI 버전과 **동일 run 포맷**을 내는지 대조 검증.
2. orchestrate/repair 순으로 치환. detect/deliver(git 조작)는 셸 그대로 두거나 마지막에.
3. 두 엔진을 `config.yaml`의 `engine: cli | sdk` 스위치로 병행 운영하며 A/B 비교.
4. 안정화 후 CLI 경로 제거.

---

## 11. 착수 전 확인 필요 사항 ⚠️

구현 시작 전에 아래를 결정해 주세요.

1. ~~**`public-python-server`가 git repo가 아닙니다.**~~ → ✅ **결정: `git init` + GitHub push**. 세 타깃이 모두 `repo: true`가 되어 파이프라인에 예외 케이스가 없어진다.
   - **선행 조건**: 파이프라인 착수 전에 초기화·push가 끝나 있어야 한다 (부록 A-1).
   - 기존 `.gitignore` 확인 결과 `.env`, `.venv/`, `.mypy_cache/`, `.pytest_cache/`, `.ruff_cache/`, `.coverage`가 모두 제외되어 있어 **첫 커밋에서 시크릿이 새어 나갈 위험은 낮다**. `.env.example`은 제외되지 않아 그대로 커밋된다 (의도한 동작).

2. ~~**매일 실행 시각**~~ → ✅ **결정: 매일 01:00 (KST)**. 시스템 타임존이 `Asia/Tokyo`지만 KST와 오프셋이 같아 그대로 `Hour: 1`로 설정하면 된다. (§7.1)

3. ~~**QA 스테이지에서 E2E를 포함할까요?**~~ → ✅ **결정: 1단계는 유닛 테스트만.** `pnpm test:e2e:all`과 pytest `integration` 마커는 Docker 기동이 전제라 무인 실행에서 가장 잘 깨진다. 파이프라인이 안정화된 뒤 config에서 `extra_gates`를 켜는 방식으로 추가한다. (§5, §6.3)

4. ~~**`specflow` 엔진 위치**~~ → ✅ **결정: GitHub private repo, 이름 `specflow`**. `~/.specflow`가 그 clone이 된다. 목적은 **새 컴퓨터 설치 편의**이므로 엔진(`bin`/`lib`/`prompts`/`templates`/`examples`/`detectors`)만 담고, 프로젝트별 `config.yaml`·`state.json`·`runs/`는 커밋하지 않는다. (§3.1~3.3)

5. ~~**spec 문서 위치**~~ → ✅ **결정: `docs/specs/`**. 기존 `docs/` 루트의 다른 문서와 섞이지 않게 전용 디렉토리를 쓴다. (§4.1)

6. ~~**첫 실 spec을 무엇으로 할까요?**~~ → ✅ **SPEC-001 원화 금액 표시 통일** 로 P8 리허설 완료 (2026-08-17). `formatWon` 유틸을 백엔드·프론트에 각각 추가하는 작고 검증하기 쉬운 과제였고, `-0.5 → "0원"` 이라는 함정을 심어 에이전트가 경계를 실제로 다루는지 확인했다. 양쪽 구현 모두 통과했다.

9. ✅ **주 에이전트를 무엇으로 할까요?** → **`agy`(Antigravity) 주력, 스테이지·타깃 단위로 교체 가능** (§12). `agy`는 호출 규약이 `claude`와 달라 설정 계층이 필요하다.

10. ✅ **기획 결과를 바로 개발까지 이을까요?** → **기본은 `draft`로 멈춘다.** `--auto-approve`를 명시할 때만 `ready`로 승격해 이어간다 (§13).

7. ~~**하룻밤 spec 처리 개수**~~ → ✅ **결정: 3개** (`limits.max_specs_per_run: 3`). 운영해보고 조정한다.

8. ~~**01:00에 맥이 잠들어 있을 텐데**~~ → ✅ **결정: `sudo pmset repeat wake MTWRFSU 00:55:00`**. 맥이 상시 AC 전원 연결 상태이고 기존 repeat 예약도 없어 안전하다. (§7.1)

---

## 12. 에이전트 계층 — Antigravity 주력 + 교체 가능 (P9)

> **결정 (2026-08-17)**: 주 에이전트를 `agy`(Antigravity)로 두고, 스테이지·타깃 단위로 다른 에이전트를 지정할 수 있게 한다.

### 12.1 왜 설정 계층이 필요한가

지금은 에이전트 커맨드가 `_orchestrate.py` 의 상수 하나(`DEFAULT_AGENT_COMMAND`)이고, 바꾸는 방법은 `SPECFLOW_AGENT_COMMAND` 환경변수로 전부 덮어쓰는 것뿐이다. 이 구조로는 "기획은 A, 구현은 B" 처럼 나눠 쓸 수 없다.

게다가 §1.3 실측대로 `agy` 와 `claude` 는 **호출 규약 자체가 다르다.** `claude` 는 프롬프트를 stdin 으로 받고, `agy` 는 `-p` 의 값으로 받으며 `--add-dir` 로 작업공간을 따로 알려 줘야 한다. 커맨드 문자열 하나로는 이 차이를 담을 수 없다.

### 12.2 config 스키마

```yaml
agent:
  default: agy

  commands:
    # 커맨드는 한 줄로 쓴다. YAML 부분집합 파서가 접힘 스칼라(>-)를 지원하지 않는다.
    agy:
      command: "agy --dangerously-skip-permissions --add-dir {cwd} --print-timeout {timeout} -p {prompt}"
      prompt_via: arg          # {prompt} 자리에 인자로 넣는다

    claude:
      command: "claude -p --permission-mode acceptEdits --allowedTools \"Read,Write,Edit,Glob,Grep,Bash\""
      prompt_via: stdin        # 표준입력으로 넘긴다 (기존 동작)

    gemini:
      command: "gemini --approval-mode yolo -m gemini-3-pro -p {prompt}"
      prompt_via: arg

  # 스테이지 또는 스테이지.타깃 단위로 덮어쓴다. 흐름 매핑({})은 파서가 거부하므로
  # 비워 둘 때는 절 자체를 생략한다.
  overrides:
    plan: claude               # 기획은 claude 로
    qa.server: claude          # 이 조합만 claude 로
```

`prompt_via` 는 생략할 수 있다. 커맨드에 `{prompt}` 가 있으면 `arg`, 없으면 `stdin` 으로 읽는다. 짧게 쓸 때는 문자열 하나로도 된다.

```yaml
agent:
  commands:
    agy: "agy --dangerously-skip-permissions --add-dir {cwd} -p {prompt}"
```

**치환 자리표시자**

| 자리표시자 | 값 |
|---|---|
| `{cwd}` | 이 스테이지의 작업 디렉토리 절대경로 (= 타깃 경로) |
| `{timeout}` | `limits.stage_timeout_minutes` 를 `30m` 형태로 |
| `{prompt}` | `prompt_via: arg` 일 때만. 렌더링된 프롬프트 전문 |

`{prompt}` 는 셸 문자열이 아니라 **argv 원소 하나**로 넘긴다. 프롬프트에는 spec 전문과 이전 단계 요약이 들어가 따옴표·개행·백틱이 섞이므로, 셸을 거치면 반드시 깨진다.

**해석 순서** (먼저 걸리는 것이 이긴다)

1. `SPECFLOW_AGENT_COMMAND` 환경변수 — 원본 커맨드를 전부 지정하는 비상구. 지금 동작을 그대로 유지한다
2. `SPECFLOW_AGENT` 환경변수 — `commands` 의 키 이름. 한 번 돌릴 때만 바꿔 보는 용도
3. `agent.overrides["<스테이지>.<타깃>"]`
4. `agent.overrides["<스테이지>"]`
5. `agent.default`

`agent:` 절이 없는 config 는 지금처럼 `claude` 로 동작한다. 이미 쓰고 있는 설정이 깨지지 않아야 한다.

### 12.3 무동작 감지 — 이번 실측이 요구한 안전장치

`agy` 는 아무것도 하지 않고도 "완료했습니다" 라고 답하며 종료 코드 0 을 낸다. 에이전트의 자기 보고를 믿지 않는다는 원칙은 게이트에 이미 적용돼 있지만, **아무것도 안 한 스테이지는 게이트도 통과한다.** 망가뜨린 것이 없기 때문이다.

P8 에서 추가한 `advanced`(브랜치 HEAD 가 움직였는가)와 `changed_files` 로 이것을 잡는다.

```
에이전트 종료 코드 0
  ├─ changed_files 있음 또는 advanced == true  → 정상
  └─ 둘 다 없음
       ├─ 요약 파일도 안 남겼다  → 스테이지 실패로 처리. 자가수정을 태우지 않고 즉시 중단
       └─ 요약은 남겼다          → 경고. "변경 없음" 으로 기록하고 다음 스테이지로
```

요약도 변경도 없는 것은 에이전트가 프롬프트를 이해하지 못했거나 도구 권한에 막힌 상태다. 그대로 다음 스테이지로 넘기면 뒤 단계가 없는 코드를 전제로 작업한다.

### 12.4 `specflow doctor` 확장

에이전트를 바꿔 끼울 수 있게 되면 "설정한 에이전트가 이 머신에 있는가" 가 새로운 실패 지점이 된다. 게이트 기준선을 재듯 여기도 미리 확인한다.

- `commands` 의 각 항목에 대해 첫 낱말이 PATH 에 있는지 (`command -v`)
- `overrides` 가 가리키는 키가 `commands` 에 실제로 있는지
- `default` 가 `commands` 에 있는지
- 자리표시자 검사 — `prompt_via: arg` 인데 `{prompt}` 가 없으면 프롬프트가 사라진다. `prompt_via: stdin` 인데 `{prompt}` 가 있으면 중복 전달된다

### 12.5 이 절이 만드는 위험

| 위험 | 대응 |
|---|---|
| `--dangerously-skip-permissions` 상시 사용 | `agy` 는 이 플래그 없이 무인 실행이 불가능하다. 대신 작업공간을 `--add-dir {cwd}` 로 **타깃 경로 하나만** 열어 사고 범위를 그 저장소로 묶는다 |
| 에이전트마다 결과 품질이 다름 | run 디렉토리에 어떤 에이전트를 썼는지 기록하고 리포트에 표기한다. 품질 문제를 에이전트 탓으로 돌릴 근거가 남는다 |
| 모델명이 config 에 하드코딩 | `agy models` 출력이 바뀌면 조용히 실패한다. doctor 가 모델 목록에 있는지도 확인한다 |

---

## 13. 기획 스테이지 — spec 문서 자동 생성 (P10)

> **결정 (2026-08-17)**: 기획 스테이지는 `status: draft` 인 spec 을 만들고 멈춘다. `--auto-approve` 를 쓸 때만 `ready` 로 승격해 같은 run 에서 개발까지 이어간다.

### 13.1 무엇을 하는 단계인가

지금 파이프라인은 사람이 쓴 spec 이 있어야 시작한다. 기획 스테이지는 그 앞에 붙어 **기존 spec 들을 읽고 다음에 할 만한 개선을 제안하는 spec 문서 하나를 쓴다.**

범위는 **기능 개선과 편의성 개선**으로 한정한다. 새 제품이나 아키텍처 변경이 아니라, 이미 있는 것을 낫게 만드는 쪽이다. 무인 실행에서 큰 것을 손대게 두면 아침에 되돌릴 수 없는 변경을 마주한다.

### 13.2 흐름

```
                    ┌──────────────────────────────┐
                    │  0. PLAN  (신설)             │
                    │  기존 spec 전부 + 최근 커밋   │
                    │  → 개선안 spec 1건 작성       │
                    │  → status: draft             │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              │ 기본                    │ --auto-approve │
              ▼                         ▼                │
      여기서 멈춘다               status: ready 로 승격   │
      사람이 읽고 ready 로              │                 │
      바꾸면 다음 run 이 집는다          ▼                 │
                                  1. DETECT 로 진입 ──────┘
```

### 13.3 에이전트에게 주는 것과 받는 것

**입력**

- `docs/specs/*.md` 전부의 frontmatter 와 요구사항 섹션 (본문 전체가 아니라 요약 — 문서가 늘면 컨텍스트가 터진다)
- `state.json` 의 처리 이력 — 무엇이 이미 구현됐는지
- 각 타깃의 최근 커밋 30건 (`git log --oneline -30`)
- 각 타깃의 `README.md` 와 `CLAUDE.md`

**출력**

`docs/specs/<번호>-<slug>.md` 한 건. 번호는 기존 최대값 + 1 로 엔진이 미리 정해 프롬프트에 박아 준다. 에이전트가 번호를 고르게 하면 충돌한다.

**금지 조항** (프롬프트에 명시)

- 이미 있는 spec 과 겹치는 제안을 하지 않는다. 입력으로 받은 목록이 그 판단 근거다
- 코드를 고치지 않는다. 이 단계의 산출물은 문서 한 개뿐이다
- `status` 를 `ready` 로 쓰지 않는다. 승격은 사람 또는 `--auto-approve` 가 한다
- 타깃 이름은 config 에 있는 것만 쓴다

### 13.4 게이트 — 문서에도 검증이 필요하다

코드 스테이지의 게이트가 lint/build/test 라면, 기획 스테이지의 게이트는 **spec 스키마 검증**이다. 이미 만들어 둔 `_validate_spec.py` 를 그대로 쓴다.

```
생성된 문서
  ├─ frontmatter 검증 통과 + status: draft  → 성공
  ├─ 검증 실패                              → docs/specs/.rejected/ 로 옮기고 리포트에 사유
  └─ 파일이 안 만들어짐                      → 스테이지 실패 (§12.3 무동작 감지)
```

검증에 실패한 문서를 `docs/specs/` 에 남기면 다음 run 의 DETECT 가 매번 그것을 집어 "스키마 검증 실패" 를 반복 보고한다. 격리하는 이유다.

### 13.5 CLI

```bash
specflow plan                    # 기획만. spec 초안 1건을 만들고 끝
specflow plan --auto-approve     # ready 로 만들고 이어서 개발까지
specflow plan --dry-run          # 무엇을 입력으로 줄지만 출력
specflow run --plan              # 기획 → 개발을 한 번에 (스케줄이 부를 형태)
```

`specflow run` 은 기본적으로 기획을 하지 않는다. 매일 밤 자동으로 새 기능을 제안하고 구현까지 하면 spec 이 통제 없이 늘어난다.

### 13.6 한도

```yaml
limits:
  max_plans_per_run: 1           # 한 번에 spec 1건만 생성
  max_draft_specs: 5             # draft 가 이만큼 쌓이면 기획을 건너뛴다
```

`max_draft_specs` 가 핵심이다. 사람이 검토하지 않은 draft 가 쌓여 있다는 것은 기획이 소비되지 않고 있다는 뜻이므로, 더 만들 이유가 없다.

### 13.7 이 절이 만드는 위험

| 위험 | 대응 |
|---|---|
| 매일 밤 쓸모없는 spec 이 쌓임 | `max_draft_specs` 로 상한. 기본은 기획을 안 돌리고 명시적으로 부를 때만 |
| 기존 spec 과 중복 제안 | 프롬프트에 기존 목록 전체를 넣고 금지 조항으로 명시. 그래도 겹치면 사람이 draft 단계에서 버린다 |
| `--auto-approve` 로 밤새 엉뚱한 기능 구현 | 기본값이 아니다. 플래그를 명시할 때만 동작한다 |
| 기획이 점점 커짐 | 프롬프트에서 범위를 "기능 개선·편의성 개선" 으로 한정하고, 비요구사항 섹션을 반드시 채우게 한다 |

---

## 부록 A. 사전 준비

**엔진 쪽**
- [ ] GitHub **private repo `specflow`** 생성
- [ ] `git clone <repo> ~/.specflow` → `install.sh` 실행
- [ ] `gh` CLI 설치 및 인증 (`gh auth status`) — PR 생성에 필요
- [ ] repo에 토큰·비밀정보·개인 경로가 들어가지 않는지 확인 (§3.2)

**이 프로젝트 쪽**
- [ ] **A-1. `public-python-server` git 초기화 + push** (§11-1 결정 사항, 파이프라인 착수 전 필수)
  - [ ] `git init` → 첫 커밋 전 `git status`로 `.env`·`.venv/`가 제외됐는지 **눈으로 확인**
  - [ ] GitHub repo 생성 (private/public 여부 결정) → `git remote add origin` → push
  - [ ] 기본 브랜치명을 다른 두 repo와 맞출 것 (`main` 권장)
- [ ] 각 repo의 `base_branch` 확인 (main / master / develop)
- [ ] `specflow init` 실행 → `.specflow/config.yaml` 생성·검토
- [ ] `.specflow/` 는 커밋 대상이 아니므로, 루트를 git repo로 만들 경우 `.gitignore`에 추가
- [ ] 첫 spec 작성 (`docs/specs/`)

> §5의 config.yaml 초안이 곧 `examples/multi-repo.yaml`의 원본이 된다. 여기서 실제 경로와 repo 이름만 자리표시자로 바꿔 동봉한다.
