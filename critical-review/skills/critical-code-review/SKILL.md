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
