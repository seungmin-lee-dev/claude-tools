# critical-review 플러그인 설계

작성일: 2026-08-18
대상 레포: `seungmin-lee-dev/claude-tools`

## 배경

`critical-code-review`는 현재 Codex 전용 스킬이다. `codex-review` 플러그인의
`setup-codex-review`가 셋업 시 `~/.codex/skills/`로 복사해주는 형태로만 존재한다.

Codex 구독이 종료되어 Codex 경로 전체(`codex-loop`, `critical-code-review`)를
실행할 수 없는 상태다. 리뷰 기준 자체는 계속 쓰고 싶으므로 Claude Code에서
단독으로 도는 형태가 필요하다.

`codex-loop`의 원래 목적은 무거운 리뷰 추론을 Codex 프로세스로 넘겨 클로드 토큰을
아끼는 것이었다. Codex가 빠지면 그 비용이 클로드로 돌아온다. 이는 설계로 제거할 수
없는 대가이며, 아래 서브에이전트 구조는 총 사용량이 아니라 **메인 대화 컨텍스트**를
보호한다.

## 결정 사항

| 항목 | 결정 |
|------|------|
| 목적 | Codex 없이 Claude 단독 리뷰 |
| 실행 형태 | 서브에이전트 1개에 위임, 최종 리포트만 회수 |
| 배포 | `claude-tools` 마켓플레이스에 신규 플러그인 분리 |
| 정책 공유 | 공유하지 않음. 정본 1개, 동기화 기계장치 없음 |
| `codex-review` | 동결. 파일 수정 없음 |
| 범위 | 리뷰 스킬 1개만. 루프는 후속 |

### 정책을 공유하지 않는 이유

초기 설계는 `shared/` 정본 + sync 스크립트 + 불일치 감지 CI였다. 이는 Codex 경로와
Claude 경로가 **둘 다 살아있다**는 전제에서 나온 구조다. Codex 경로가 실행되지 않는
현재, Codex 쪽 사본은 아무도 읽지 않는 파일이며 그것과의 동기화를 CI로 강제하는 것은
갈라져도 아무 결과가 없는 상태를 지키느라 매 PR마다 비용을 내는 것이다.

구독 재개 시 정본(`critical-review` 쪽)에서 Codex 쪽으로 반영하면 된다.

## 구조

```
claude-tools/
├── .claude-plugin/marketplace.json     # plugins: codex-review, critical-review
│
├── critical-review/                    # 신규
│   ├── .claude-plugin/plugin.json
│   ├── README.md
│   ├── agents/
│   │   └── critical-code-reviewer.md   # 정책 정본
│   └── skills/critical-code-review/
│       └── SKILL.md                    # 오케스트레이터
│
└── codex-review/                       # 동결
    └── README.md                       # 정본 위치 한 줄 추가
```

## 컴포넌트

### `agents/critical-code-reviewer.md` — 리뷰 정책 정본

**역할:** 리뷰 판단 기준 전체를 담는다. 서브에이전트의 시스템 프롬프트가 된다.

**본문:** 기존 Codex `critical-code-review/SKILL.md`의 본문을 그대로 이식한다.
해당 본문은 이미 도구 중립적이다 — Codex 고유 명령이나 경로 참조가 없고, 순수하게
리뷰 정책, 패널 구성, 증거 규칙, 심각도 기준, 출력 포맷만 담고 있다. 내용 수정 없음.

**frontmatter:** Claude 에이전트용으로 새로 작성한다.

```yaml
name: critical-code-reviewer
description: 구현된 코드, diff, PR을 시니어/QA/CTO 패널 관점으로 비판적으로 리뷰한다. 승인을 기본값으로 두지 않는다.
tools: Read, Grep, Glob, Bash
```

`tools`에 조사 도구를 주는 이유: 정책의 Evidence Rules가 "인접 모듈, 직접 호출자,
consumers, imports, 테스트를 조사한 뒤 구조적 판단을 내리라"고 요구한다. 도구가
없으면 이 항목들이 실행 불가능한 문장이 된다.

쓰기 도구(Edit, Write)는 주지 않는다. 리뷰는 코드를 수정하지 않는다.

**정책을 에이전트 정의에 두는 이유:** `policy.md`를 두고 SKILL.md가 읽어서 프롬프트에
싣는 방법도 가능하지만, 그러면 긴 정책 본문이 메인 컨텍스트에 한 번 올라와 서브에이전트로
격리한 목적이 반감된다. 에이전트 정의 파일에 두면 정책은 서브에이전트 쪽에만 로드된다.

### `skills/critical-code-review/SKILL.md` — 오케스트레이터

**역할:** 리뷰 대상을 확보해 에이전트에 넘기고 결과를 전달한다. 리뷰 판단은 하지 않는다.

**동작:**

1. **인자 파싱**
   - 인자 없음 → uncommitted diff (`git diff`)
   - `<PR번호>` → `gh pr diff <N>`
   - `<브랜치>` → `git diff main...<branch>` (base는 레포 기본 브랜치)
   - `<경로>` → 해당 파일/디렉터리
   - `--detailed` → Detailed Mode
   - `--structural` → Structural Deep-Dive Mode

2. **전제 확인** — git 레포인지 확인. PR 대상이면 `gh` 존재 확인.

3. **대상 확보** — diff가 비어 있으면 "리뷰할 변경 없음"으로 종료.

4. **에이전트 호출** — `subagent_type: critical-code-reviewer`
   - 프롬프트: 리뷰 대상(diff 또는 경로) + 모드 + 레포 직접 조사 허용 명시
   - diff가 대략 6만 토큰을 넘으면 파일 단위로 나눠 호출하거나 범위 축소를 제안 (codex-loop과 동일 기준)

5. **출력** — 회수한 리포트를 **요약 없이 그대로** 전달한다.

5단계가 중요하다. 메인이 리포트를 재요약하면 findings가 소실되고 severity 구분이
뭉개진다. 정책이 규정한 출력 포맷(Findings / High-Risk Areas / Missing Tests /
전문가 토론 요약 / Required Fixes / Final Review Decision)을 그대로 통과시킨다.

### `plugin.json` / `marketplace.json`

`critical-review` 플러그인을 마켓플레이스 `plugins` 배열에 추가한다.
초기 버전 `0.1.0`.

### `codex-review/README.md`

정책 정본이 `critical-review` 플러그인으로 이동했고 이 플러그인은 동결 상태임을
한 줄 명시한다. 그 외 파일은 수정하지 않는다.

## 이름

- 플러그인: `critical-review`
- 스킬: `critical-code-review` (Codex 쪽과 동일 — 같은 기능)
- 에이전트: `critical-code-reviewer`
- 호출: `/critical-review:critical-code-review`

Claude Code 내장 `/code-review`, gstack의 `/review`, `/pr-review`와 트리거가
겹칠 수 있다. 스킬 description에 "한국어 전문가 패널 리뷰"를 명시해 구분한다.

## 검증

플러그인 스킬은 자동 테스트 대상이 아니므로 수동 검증으로 확인한다.

1. `/plugin marketplace add` → `/plugin install critical-review` 로 설치되는지
2. 실제 uncommitted diff에 대해 `/critical-review:critical-code-review` 실행
3. 확인 항목:
   - 서브에이전트가 호출되고 메인 컨텍스트에 정책 본문이 올라오지 않는지
   - 에이전트가 diff 밖 파일(호출자, 테스트)을 실제로 조사하는지
   - 출력이 정책의 Compact Output Format을 따르는지
   - `Final Review Decision`이 4개 verdict 중 하나로 나오는지
   - 코드가 수정되지 않는지 (읽기 전용 확인)
4. `--detailed`, `--structural`, PR 번호 인자 각각 1회 실행

## 범위 밖 (후속 작업)

- **`review-loop`** — codex-loop의 Claude판(리뷰 → 수정 → 재리뷰 반복).
  리뷰 스킬 품질이 검증된 뒤 그 위에 얹는다.
- **레포에 없는 로컬 Codex 스킬 3종** — `critical-design-review`,
  `claude-implementation-handoff`, `mindon-issue`가 `~/.codex/skills/`에만 존재하고
  레포에 반영돼 있지 않다. 별도 이슈로 처리.
- **`~/.claude/skills/codex-loop` 개인 사본 정리** — 플러그인 설치로 대체 가능.
