# critical-review 플러그인 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Codex 전용이던 `critical-code-review` 리뷰 기준을 Claude Code 단독으로 실행되는 신규 플러그인 `critical-review`로 제공한다.

**Architecture:** 플러그인 루트에 `agents/critical-code-reviewer.md`(리뷰 정책 정본 = 서브에이전트 시스템 프롬프트)를 두고, `skills/critical-code-review/SKILL.md`는 리뷰 대상만 확보해 그 에이전트에 위임한 뒤 리포트를 그대로 통과시키는 얇은 오케스트레이터로 둔다. 정책 본문이 메인 대화 컨텍스트에 로드되지 않는 것이 이 분리의 목적이다.

**Tech Stack:** Claude Code 플러그인 (marketplace.json / plugin.json / agents / skills), Markdown + YAML frontmatter. 빌드 도구 없음.

## Global Constraints

- 대상 레포: `seungmin-lee-dev/claude-tools`, 작업 브랜치 `feat/critical-review-plugin`
- 레포 절대 경로: `C:\Users\dsnlab-lsm\Desktop\MindOn\claude-tools` (bash에서는 `/c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools`)
- 스펙: `docs/superpowers/specs/2026-08-18-critical-review-plugin-design.md`
- `codex-review/` 는 **동결**. `codex-review/README.md` 한 줄 추가를 제외하고 어떤 파일도 수정하지 않는다.
- 리뷰 정책 본문은 **수정하지 않는다.** 원본 `codex-review/skills/setup-codex-review/codex-skills/critical-code-review/SKILL.md` 의 5~253행(249행)을 그대로 옮긴다. 서브에이전트용 지시(출력 계약, 읽기 전용 등)는 정책에 넣지 않고 오케스트레이터의 호출 프롬프트에 둔다.
- 플러그인 초기 버전: `0.1.0` (`plugin.json`과 `marketplace.json` 양쪽 동일)
- 에이전트 이름: `critical-code-reviewer` / 호출 시 `subagent_type`: `critical-review:critical-code-reviewer`
- 스킬 이름: `critical-code-review` / 호출 시 `/critical-review:critical-code-review`
- 에이전트 `tools`: `Read, Grep, Glob, Bash` — 쓰기 도구(Edit, Write)는 주지 않는다
- 문서·설명은 한국어

## File Structure

| 파일 | 책임 |
|------|------|
| `critical-review/.claude-plugin/plugin.json` | 플러그인 메타데이터 |
| `.claude-plugin/marketplace.json` (수정) | 마켓플레이스에 플러그인 등록 |
| `critical-review/agents/critical-code-reviewer.md` | **리뷰 정책 정본.** 서브에이전트 시스템 프롬프트 |
| `critical-review/skills/critical-code-review/SKILL.md` | 오케스트레이터. 대상 확보 → 위임 → 통과 |
| `critical-review/README.md` | 플러그인 사용법 |
| `README.md` (수정) | 루트 플러그인 목록·구조 갱신 |
| `codex-review/README.md` (수정) | 동결 상태 및 정본 위치 명시 |

---

### Task 1: 플러그인 뼈대와 마켓플레이스 등록

설치 가능한 빈 플러그인을 먼저 만든다. 이후 태스크가 여기에 내용물을 채운다.

**Files:**
- Create: `critical-review/.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

**Interfaces:**
- Consumes: 없음 (첫 태스크)
- Produces: 플러그인 식별자 `critical-review@claude-tools`. 이후 모든 파일은 `critical-review/` 아래에 놓인다.

- [ ] **Step 1: 플러그인 매니페스트 생성**

`critical-review/.claude-plugin/plugin.json` 을 아래 내용으로 만든다.

```json
{
  "name": "critical-review",
  "description": "Claude 단독으로 도는 비판적 코드 리뷰. 구현된 코드/diff/PR을 시니어·QA·회의적 리뷰어·CTO 패널 관점으로 검토하고 한국어 findings와 최종 verdict를 낸다. 외부 CLI 의존 없음.",
  "version": "0.1.0",
  "author": {
    "name": "MindOn"
  },
  "keywords": [
    "review",
    "code-review",
    "korean",
    "qa"
  ]
}
```

- [ ] **Step 2: 마켓플레이스에 등록**

`.claude-plugin/marketplace.json` 의 `plugins` 배열에 두 번째 항목을 추가한다. 파일 전체가 아래와 같이 되어야 한다.

```json
{
  "name": "claude-tools",
  "description": "팀 내부 Claude Code 도구 모음 (플러그인 마켓플레이스)",
  "owner": {
    "name": "seungmin-lee-dev"
  },
  "plugins": [
    {
      "name": "codex-review",
      "source": "./codex-review",
      "description": "Codex CLI 기반 리뷰·문서 오프로드 워크플로 (codex-loop, setup-codex-review)",
      "version": "0.3.1",
      "author": {
        "name": "seungmin-lee-dev"
      }
    },
    {
      "name": "critical-review",
      "source": "./critical-review",
      "description": "Claude 단독 비판적 코드 리뷰 (critical-code-review). 외부 CLI 불필요",
      "version": "0.1.0",
      "author": {
        "name": "seungmin-lee-dev"
      }
    }
  ]
}
```

- [ ] **Step 3: JSON 유효성 검증**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
python -c "
import json
m=json.load(open('.claude-plugin/marketplace.json',encoding='utf-8'))
p=json.load(open('critical-review/.claude-plugin/plugin.json',encoding='utf-8'))
e=[x for x in m['plugins'] if x['name']=='critical-review'][0]
assert len(m['plugins'])==2, m['plugins']
assert e['version']==p['version'], (e['version'], p['version'])
print('OK', p['name'], p['version'])
"
```

Expected: `OK critical-review 0.1.0`

`node` 는 이 환경(Git Bash)에서 복잡한 따옴표와 함께 쓰면 cmd.exe 오류가 난다. python 을 쓴다.
파이썬 출력에 한글이 섞이면 앞에 `PYTHONIOENCODING=utf-8` 을 붙인다.

버전 불일치나 JSON 문법 오류면 여기서 실패한다. 실패 시 Step 1~2를 고치고 재실행.

- [ ] **Step 4: 커밋**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
git add .claude-plugin/marketplace.json critical-review/.claude-plugin/plugin.json
git commit -m "feat(critical-review): 플러그인 뼈대와 마켓플레이스 등록"
```

---

### Task 2: 리뷰 에이전트 정의 (정책 정본)

**Files:**
- Create: `critical-review/agents/critical-code-reviewer.md`
- 읽기 전용 참조: `codex-review/skills/setup-codex-review/codex-skills/critical-code-review/SKILL.md`

**Interfaces:**
- Consumes: Task 1의 `critical-review/` 디렉터리
- Produces: 서브에이전트 타입 `critical-review:critical-code-reviewer`. Task 3의 SKILL.md가 이 이름으로 호출한다.

**주의:** 정책 본문 249행을 사람이 옮겨 적지 않는다. 손으로 복사하면 조용한 누락이 생긴다. 아래 명령으로 추출·결합한다.

- [ ] **Step 1: 정책 본문 추출**

원본은 1~4행이 frontmatter, 5~253행이 본문이다. 파일 전체에서 `---` 줄은 1행과 4행 둘뿐이므로 5행부터 끝까지가 정확히 본문이다.

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
SRC=codex-review/skills/setup-codex-review/codex-skills/critical-code-review/SKILL.md
grep -n '^---$' "$SRC"
tail -n +5 "$SRC" > /tmp/policy-body.md
wc -l < /tmp/policy-body.md
```

Expected:
```
1:---
4:---
249
```

`---` 위치가 1과 4가 아니거나 라인 수가 249가 아니면 원본이 바뀐 것이다. 진행하지 말고 실제 frontmatter 끝 행 번호 N을 확인해 `tail -n +$((N+1))` 로 조정한다.

- [ ] **Step 2: frontmatter를 붙여 에이전트 파일 생성**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
mkdir -p critical-review/agents
cat > /tmp/agent-frontmatter.md <<'FM'
---
name: critical-code-reviewer
description: 구현된 코드, diff, PR, 패치를 정확성·회귀·테스트·보안·운영 준비도·코드 구조·모듈화·성능 관점에서 비판적으로 리뷰한다. 시니어/QA/회의적 리뷰어/CTO 패널로 동작하며 승인을 기본값으로 두지 않는다. critical-code-review 스킬이 위임할 때 사용.
tools: Read, Grep, Glob, Bash
---
FM
cat /tmp/agent-frontmatter.md /tmp/policy-body.md > critical-review/agents/critical-code-reviewer.md
```

- [ ] **Step 3: 결과 검증**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
AGENT=critical-review/agents/critical-code-reviewer.md
SRC=codex-review/skills/setup-codex-review/codex-skills/critical-code-review/SKILL.md
echo "총 라인: $(wc -l < $AGENT) (기대: 254)"
sed -n '1,5p' "$AGENT"
echo "--- 본문 동일 여부 ---"
diff <(tail -n +6 "$AGENT") <(tail -n +5 "$SRC") && echo "본문 IDENTICAL"
```

Expected:
- `총 라인: 254 (기대: 254)` — frontmatter 5행 + 본문 249행
- 1~5행이 `---` / `name:` / `description:` / `tools:` / `---`
- `본문 IDENTICAL`

`본문 IDENTICAL` 이 안 나오면 정책이 변형된 것이다. 반드시 여기서 멈추고 Step 1~2를 다시 실행한다.

- [ ] **Step 4: 커밋**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
git add critical-review/agents/critical-code-reviewer.md
git commit -m "feat(critical-review): 리뷰 정책 에이전트 정의 추가"
```

---

### Task 3: 오케스트레이터 스킬

**Files:**
- Create: `critical-review/skills/critical-code-review/SKILL.md`

**Interfaces:**
- Consumes: Task 2의 에이전트 타입 `critical-review:critical-code-reviewer`
- Produces: 슬래시 커맨드 `/critical-review:critical-code-review`

- [ ] **Step 1: SKILL.md 작성**

`critical-review/skills/critical-code-review/SKILL.md` 를 아래 내용 그대로 만든다.

~~~markdown
---
name: critical-code-review
description: 구현된 코드, diff, PR, 브랜치를 한국어 전문가 패널(Staff/QA/회의적 시니어/CTO) 관점으로 비판적으로 리뷰한다. 승인을 기본값으로 두지 않고 확정 버그를 먼저 찾으며 Critical/High/Medium/Low 심각도와 최종 verdict를 낸다. Codex 등 외부 CLI 없이 Claude 단독으로 동작. "크리티컬 리뷰", "코드 리뷰해줘", "배포 전 검토", "모듈화 검토", "리팩터링 검증", "critical review" 요청 시 사용.
---

# critical-code-review

리뷰 대상을 확보해 `critical-review:critical-code-reviewer` 서브에이전트에 위임하고,
돌아온 리포트를 **그대로** 전달한다. 이 스킬 자체는 리뷰 판단을 하지 않는다.

리뷰 정책 전문은 에이전트 정의(`agents/critical-code-reviewer.md`)에 있다.
**그 파일을 읽지 마라.** 정책을 메인 컨텍스트에서 격리하는 것이 이 구조의 목적이다.

## 1. 인자 파싱

| 인자 | 리뷰 대상 |
|------|-----------|
| 없음 | uncommitted diff (없으면 staged diff) |
| 숫자 (예: `12`) | 해당 PR의 diff |
| 브랜치명 | 기본 브랜치와의 diff |
| 경로 | 해당 파일/디렉터리 전체 |

플래그:
- `--detailed` → Detailed Mode (전문가별 상세 비평)
- `--structural` → Structural Deep-Dive Mode (아키텍처·모듈화·리팩터링 계획)
- 없으면 Compact Mode (기본)

## 2. 전제 확인

```bash
git rev-parse --is-inside-work-tree
```

git 레포가 아니면 알리고 종료한다. PR 번호가 주어졌으면 `gh --version` 도 확인한다.

## 3. 대상 확보

- **인자 없음:**
  ```bash
  git diff
  ```
  비어 있으면 `git diff --cached` 를 확인한다. 둘 다 비어 있으면
  "리뷰할 변경이 없습니다"를 알리고 종료한다.

- **PR 번호:**
  ```bash
  gh pr diff <번호>
  ```

- **브랜치명:**
  ```bash
  BASE=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|origin/||')
  BASE=${BASE:-main}
  git diff "$BASE...<브랜치명>"
  ```

- **경로:** diff가 아니라 해당 파일/디렉터리 전체를 리뷰 대상으로 삼는다.
  내용을 여기서 읽지 말고 경로만 에이전트에 넘겨 직접 읽게 한다.

**크기 확인:**

```bash
git diff | wc -c
```

240000(약 6만 토큰)을 넘으면 파일 단위로 나눠 여러 번 호출하거나 범위를 좁히자고
사용자에게 제안한다. 임의로 잘라내지 마라.

## 4. 에이전트 호출

Agent 도구를 `subagent_type: critical-review:critical-code-reviewer` 로 호출한다.
프롬프트는 아래 형식을 따른다.

```
리뷰 대상: <무엇인지 한 줄. 예: "현재 uncommitted diff" / "PR #12" / "src/auth/ 디렉터리">
모드: <Compact | Detailed | Structural Deep-Dive>
레포 루트: <절대 경로>

<diff 본문, 또는 경로 목록>

지시:
- 변경분에 한정해 리뷰하라. 변경과 무관한 기존 코드는 지적하지 마라.
- 필요하면 Read/Grep/Glob/Bash로 인접 모듈, 직접 호출자, consumers, 테스트,
  아키텍처 문서를 직접 조사하라. 근거 없는 구조 판단은 하지 마라.
- 코드를 수정하지 마라. 읽기 전용이다.
- 네 최종 출력이 그대로 사용자에게 전달된다. 인사말·서두·맺음말 없이
  정책이 규정한 출력 포맷만 출력하라.
```

## 5. 출력

에이전트가 돌려준 리포트를 **요약하거나 재구성하지 말고 그대로 출력한다.**

메인이 다시 요약하면 findings가 소실되고 심각도 구분이 뭉개진다.
스킬이 덧붙일 수 있는 것은 리포트 앞뒤의 한 줄짜리 맥락(무엇을 리뷰했는지)뿐이다.

## 레드 플래그

- 에이전트 정의 파일을 Read로 열었다 → 격리 실패. 열지 마라.
- 리포트를 "요약하면…" 으로 다시 쓰고 있다 → 그대로 통과시켜라.
- diff가 비었는데 레포 전체를 리뷰하고 있다 → 종료했어야 한다.
- 리뷰 결과를 받고 코드를 고치기 시작했다 → 이 스킬의 범위가 아니다.
  수정은 사용자가 별도로 요청해야 한다.
~~~

- [ ] **Step 2: frontmatter 검증**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
SK=critical-review/skills/critical-code-review/SKILL.md
sed -n '1,4p' "$SK"
echo "--- 에이전트 참조 횟수 ---"
grep -c 'critical-review:critical-code-reviewer' "$SK"
```

Expected:
- 1행 `---`, 2행 `name: critical-code-review`, 3행 `description: ...`, 4행 `---`
- 에이전트 참조 횟수 `2`

- [ ] **Step 3: 에이전트 이름 일치 확인**

Task 2에서 정의한 이름과 SKILL.md가 부르는 이름이 같아야 한다. 다르면 런타임에 에이전트를 못 찾는다.

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
grep '^name:' critical-review/agents/critical-code-reviewer.md
grep -o 'critical-review:critical-code-reviewer' critical-review/skills/critical-code-review/SKILL.md | head -1
```

Expected: `name: critical-code-reviewer` 와 `critical-review:critical-code-reviewer` — 콜론 뒤 이름이 일치.

- [ ] **Step 4: 커밋**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
git add critical-review/skills/critical-code-review/SKILL.md
git commit -m "feat(critical-review): 리뷰 오케스트레이터 스킬 추가"
```

---

### Task 4: 문서 (플러그인 README + 루트 README + codex-review 동결 표시)

**Files:**
- Create: `critical-review/README.md`
- Modify: `README.md`
- Modify: `codex-review/README.md`

**Interfaces:**
- Consumes: Task 1~3의 플러그인/스킬/에이전트 이름
- Produces: 없음 (문서)

- [ ] **Step 1: 플러그인 README 작성**

`critical-review/README.md` 를 아래 내용으로 만든다.

~~~markdown
# critical-review (Claude Code 플러그인)

구현된 코드·diff·PR을 **시니어 / QA / 회의적 리뷰어 / CTO 패널** 관점으로 비판적으로 리뷰한다.
승인을 기본값으로 두지 않고, 확정 버그를 개선 제안보다 먼저 찾는다.

**외부 CLI 의존이 없다.** Codex 설치·로그인 없이 Claude Code만으로 동작한다.

## 포함 스킬

- **`/critical-review:critical-code-review`** — 리뷰 실행
  - 인자 없음: 현재 uncommitted diff (없으면 staged)
  - `<PR번호>` / `<브랜치명>` / `<경로>`
  - 플래그: `--detailed`(전문가별 상세 비평), `--structural`(아키텍처·모듈화 심층)

자연어로 "크리티컬 리뷰해줘", "배포 전 검토해줘"처럼 요청해도 트리거된다.

## 설치

```
/plugin marketplace add seungmin-lee-dev/claude-tools
/plugin install critical-review@claude-tools
```

설치 후 아무 레포/경로에서 사용 가능.

## 동작 구조

리뷰는 전용 서브에이전트(`critical-code-reviewer`)에 위임된다.
긴 리뷰 정책과 패널 추론이 메인 대화 컨텍스트에 올라오지 않고, 최종 리포트만 돌아온다.

에이전트는 `Read, Grep, Glob, Bash` 만 갖는다. 정책이 "인접 모듈·호출자·테스트를
조사한 뒤 구조를 판단하라"고 요구하기 때문이다.
**쓰기 도구는 없다 — 리뷰는 코드를 고치지 않는다.**

## 출력

```
## Findings                (Critical/High/Medium/Low 순, 파일·라인 근거 포함)
## High-Risk Areas
## Missing Tests
## 전문가 토론 요약
## Required Fixes
## Follow-up Improvements
## Final Review Decision   (Reject | Needs Revision | Conditionally Accept | Accept)
```

## 비용

리뷰 추론이 클로드 쪽에서 돈다. 서브에이전트 격리는 **메인 대화 컨텍스트**를 보호하는 것이지
총 토큰 사용량을 줄이지는 않는다. 큰 diff는 범위를 좁혀 여러 번 돌리는 편이 낫다.

## 리뷰 정책 수정

정책 정본은 [`agents/critical-code-reviewer.md`](agents/critical-code-reviewer.md) 다.
같은 정책의 Codex용 사본이 `codex-review` 플러그인에 있으나 현재 **동결** 상태이며 동기화하지 않는다.

## 업데이트

push 후 팀원은 `/plugin`에서 업데이트. `plugin.json`과 루트 `marketplace.json`의 `version`을 함께 올린다.
~~~

- [ ] **Step 2: 루트 README에 플러그인 추가**

`README.md` 의 `## 플러그인 목록` 섹션에서 `### codex-review` 블록 **뒤에** 아래를 삽입한다.

```markdown
### critical-review
Claude 단독 비판적 코드 리뷰. 외부 CLI 불필요 — 설치하면 바로 사용 가능.
- 설치: `/plugin install critical-review@claude-tools`
- 스킬: `/critical-review:critical-code-review`
- 자세한 내용: [`critical-review/README.md`](critical-review/README.md)
```

같은 파일 `## 구조` 의 코드 블록을 아래로 교체한다.

```
.claude-plugin/marketplace.json   # 이 레포 = 마켓플레이스
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
```

- [ ] **Step 3: codex-review 동결 표시**

`codex-review/README.md` 의 맨 위 제목(`# codex-review (Claude Code 플러그인)`) 바로 아래 줄에 아래 블록을 삽입한다. **다른 부분은 건드리지 않는다.**

```markdown
> **동결(frozen).** Codex 구독 종료로 현재 이 플러그인은 유지보수하지 않는다.
> `critical-code-review` 정책 정본은 [`critical-review/agents/critical-code-reviewer.md`](../critical-review/agents/critical-code-reviewer.md) 로 옮겨졌고,
> 여기 번들된 Codex용 사본은 동기화되지 않는다. Codex 구독을 재개하면 정본에서 이쪽으로 반영할 것.
```

- [ ] **Step 4: 링크와 동결 범위 검증**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
test -f critical-review/README.md && echo "플러그인 README OK"
grep -q 'critical-review@claude-tools' README.md && echo "루트 README OK"
grep -q '동결(frozen)' codex-review/README.md && echo "동결 표시 OK"
echo "--- codex-review 변경 범위 ---"
git status --short codex-review/
```

Expected: 세 줄 모두 `OK`, 그리고 `git status --short codex-review/` 출력이 `M codex-review/README.md` 한 줄뿐. 다른 파일이 나오면 동결 제약 위반이므로 되돌린다.

- [ ] **Step 5: 커밋**

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
git add critical-review/README.md README.md codex-review/README.md
git commit -m "docs(critical-review): 플러그인 README 및 codex-review 동결 표시"
```

---

### Task 5: 설치 및 실동작 검증

문서상 맞는 것과 실제로 도는 것은 다르다. 이 태스크는 실제로 설치해서 돌려본다.
자동 테스트 대상이 아니므로 수동 검증이 유일한 검증 수단이다.

**Files:**
- 검증 결과에 따라 Task 1~4의 파일을 수정할 수 있음

**Interfaces:**
- Consumes: Task 1~4 전부
- Produces: 동작 확인된 플러그인

- [ ] **Step 1: 로컬 마켓플레이스로 설치**

사용자에게 Claude Code 세션에서 아래 슬래시 커맨드를 직접 실행하도록 요청한다.

```
/plugin marketplace add C:\Users\dsnlab-lsm\Desktop\MindOn\claude-tools
/plugin install critical-review@claude-tools
```

설치 후 스킬 목록에 `critical-review:critical-code-review` 가, 에이전트 타입 목록에
`critical-review:critical-code-reviewer` 가 보이는지 확인한다. 안 보이면 세션을 재시작한다.

- [ ] **Step 2: 기본 모드 실행**

리뷰할 uncommitted 변경이 있는 레포에서 실행한다. 변경이 없으면 임시로 만든다.

```
/critical-review:critical-code-review
```

- [ ] **Step 3: 체크리스트 확인**

아래 6개를 모두 확인한다. 하나라도 실패하면 원인을 고치고 Step 2부터 다시.

1. 서브에이전트가 실제로 호출되었다 (메인이 직접 리뷰하지 않았다)
2. 메인 컨텍스트에 정책 본문 249행이 로드되지 않았다 — `agents/critical-code-reviewer.md` 를 Read하지 않았다
3. 에이전트가 diff 밖의 파일(호출자·테스트 등)을 실제로 조사했다
4. 출력이 정책의 Compact Output Format을 따른다 (`## Findings` 로 시작)
5. `## Final Review Decision` 에 `Reject | Needs Revision | Conditionally Accept | Accept` 중 하나가 나온다
6. 코드가 수정되지 않았다 — `git status` 가 실행 전과 동일하다

- [ ] **Step 4: 나머지 인자 형태 확인**

각각 1회씩 실행해 인자 파싱이 동작하는지 본다.

```
/critical-review:critical-code-review --detailed
/critical-review:critical-code-review --structural
/critical-review:critical-code-review <경로>
```

`--detailed` 는 전문가별 비평이, `--structural` 은 `## Structure & Modularity` 가
출력에 나타나야 한다. PR 번호 형태는 이 브랜치로 PR을 연 뒤 그 번호로 확인한다.

- [ ] **Step 5: 검증 중 발견한 수정 반영 후 푸시**

수정이 있었다면 커밋한다.

```bash
cd /c/Users/dsnlab-lsm/Desktop/MindOn/claude-tools
git add -A critical-review/
git commit -m "fix(critical-review): 실동작 검증에서 발견한 문제 수정"
```

수정이 없었다면 커밋 없이 통과. 마지막으로 브랜치를 푸시한다.

```bash
git push
```

---

## 완료 조건

- `/plugin install critical-review@claude-tools` 로 설치된다
- `/critical-review:critical-code-review` 가 Codex 없이 동작한다
- 정책 본문이 원본 249행과 동일하다 (`diff` 로 확인됨)
- `codex-review/` 변경이 `README.md` 한 건뿐이다
- 리뷰 실행이 코드를 수정하지 않는다

## 범위 밖

- `review-loop` (codex-loop의 Claude판) — 리뷰 품질 검증 후 별도 작업
- 레포에 없는 로컬 Codex 스킬 3종(`critical-design-review`, `claude-implementation-handoff`, `mindon-issue`) 반영
- `~/.claude/skills/codex-loop` 개인 사본 정리
