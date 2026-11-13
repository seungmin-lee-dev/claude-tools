# skills

Claude Code 일반 스킬 모음 (플러그인 마켓플레이스).
한 레포에 여러 플러그인을 담고, 마켓플레이스를 한 번 등록한 뒤 필요한 플러그인을 설치한다.

> 저장소 이름은 `claude-tools` → `skills` 로 바뀌었다(2026-11-13). 옛 주소는 GitHub 가 새 주소로 연결해 준다.
> **설치 이름은 그대로 `@claude-tools`** 다(마켓플레이스 이름). 이미 설치한 사람은 아무것도 안 해도 된다.

## 공개 / 비공개 나눠 관리
| 저장소 | 공개 | 담는 것 |
|---|---|---|
| `seungmin-lee-dev/skills` (여기) | 공개 | 어느 프로젝트에서나 쓰는 일반 스킬 |
| 비공개 스킬 저장소 | 비공개 | 회사 프로젝트 전용 값(주소·계정·도메인·내부 규칙)이 든 스킬 |

**회사 내부 값은 여기에 올리지 않는다.** 일반 절차는 여기에, 프로젝트 값은 비공개 저장소의 전용 스킬에 두고 전용 스킬이 일반 스킬을 불러 쓴다.

## 빠른 시작 (1회) — Claude Code 안에서 아래를 순서대로 입력
```
/plugin marketplace add seungmin-lee-dev/skills
/plugin install codex-review@claude-tools
/codex-review:setup-codex-review
```
- ①②는 자동. 설치 후 명령이 안 보이면 세션을 한 번 재시작.
- ③ 실행 시 **Codex CLI 설치 + `codex login`** 이 안 돼 있으면 안내가 뜬다 → 그 부분만 각자 1회 수동:
  - 설치(대표): `npm install -g @openai/codex`
  - 로그인: 터미널에서 `codex login`
- 이후 아무 레포/경로에서 **`/codex-review:codex-loop`** 사용 (자연어 "코덱스로 리뷰해줘"도 트리거됨).

## 플러그인 목록

스킬 이름은 `플러그인:스킬` 로 불러서 서로 구별된다.

| 플러그인 | 스킬 | 한 줄 설명 |
|---|---|---|
| `codex-review` | `/codex-review:codex-loop`, `/codex-review:setup-codex-review` | 리뷰·문서를 Codex 에 넘기는 반복 개선 루프 |
| `critical-review` | `/critical-review:critical-code-review` | Claude 단독 비판적 코드 리뷰 |
| `ego-qa` | `/ego-qa:ego-qa` | ego-browser 로 로컬 웹 QA·PR 실제 동작 시험 |

### codex-review
Codex CLI 기반 리뷰·문서 오프로드 워크플로. 코드는 클로드, 리뷰·문서는 Codex → 클로드 토큰 절감.
- 설치: `/plugin install codex-review@claude-tools`
- 자세한 내용: [`codex-review/README.md`](codex-review/README.md)

### critical-review
Claude 단독 비판적 코드 리뷰. 외부 CLI 불필요 — 설치하면 바로 사용 가능.
- 설치: `/plugin install critical-review@claude-tools`
- 자세한 내용: [`critical-review/README.md`](critical-review/README.md)

### ego-qa
ego-browser(ego lite)로 로컬 웹 앱을 직접 눌러 보며 QA. 남의 PR 을 별도 워크트리로 띄워 실제로 되는지 시험하고, 콘솔 오류·실패한 요청까지 모아 보고한다.
- 설치: `/plugin install ego-qa@claude-tools`
- 필요: [ego lite](https://github.com/citrolabs/ego-lite) 설치
- 자세한 내용: [`ego-qa/README.md`](ego-qa/README.md)

## 새 스킬(플러그인) 추가하는 법
1. 회사 내부 값이 들어가면 **여기가 아니라 비공개 저장소**에 추가한다.
2. `<플러그인명>/.claude-plugin/plugin.json` + `<플러그인명>/skills/<스킬명>/SKILL.md` 로 하위 폴더 추가.
   다른 플러그인 스킬과 헷갈리지 않게 스킬 이름은 하는 일이 드러나게 짓는다.
3. 루트 `.claude-plugin/marketplace.json`의 `plugins[]`에 `{ "name": ..., "source": "./<플러그인명>", ... }` 등록
4. 버전 올리고 push → `/plugin`에서 업데이트

## 구조
```
.claude-plugin/marketplace.json   # 이 레포 = 마켓플레이스 (이름: claude-tools)
codex-review/                     # 플러그인 (Codex 기반, 현재 동결)
  .claude-plugin/plugin.json
  skills/
    codex-loop/SKILL.md
    setup-codex-review/SKILL.md
critical-review/                  # 플러그인 (Claude 단독)
  .claude-plugin/plugin.json
  agents/critical-code-reviewer.md
  skills/
    critical-code-review/SKILL.md
ego-qa/                           # 플러그인 (ego-browser 필요)
  .claude-plugin/plugin.json
  skills/
    ego-qa/SKILL.md
```
