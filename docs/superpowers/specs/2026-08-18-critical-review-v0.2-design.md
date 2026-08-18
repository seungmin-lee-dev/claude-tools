# critical-review v0.2 — 리뷰 운영 품질 개선 설계

작성일: 2026-08-18
대상 레포: `seungmin-lee-dev/claude-tools`
선행 문서: [`2026-08-18-critical-review-plugin-design.md`](2026-08-18-critical-review-plugin-design.md)

## 배경

v0.1.0을 실제로 한 번 돌리고(계획서 Task 5), 인기 있는 Claude용 코드 리뷰
스킬들과 비교한 결과 나온 개선이다. 추측이 아니라 아래 두 가지 근거에 기반한다.

### 근거 1: 실동작 측정 (winsam PR #40)

| 항목 | 측정값 |
|------|--------|
| 대상 | 17파일 1,075 insertions (C# + 셸 + XML) |
| 소요 | 9분 29초 |
| 서브에이전트 토큰 | 105,063 |
| 툴 사용 | 31회 |
| 최종 판정 | Needs Revision |

Task 5 체크리스트 6개 중 5개 통과, 1개(메인 컨텍스트 정책 미로드)는 해당 세션에서
검증자가 비교 목적으로 정책을 먼저 읽어 검증 불가. 스킬 실행 경로 자체는 정책을
읽지 않음을 확인했다.

출력 품질은 양호했다. 재현 경로가 포함된 finding이 실제로 나왔고
(`sec\ret\` → `'sec\ret\'`, `exit 127`), diff 밖 파일을 근거로 인용했다.

### 근거 2: 경쟁 스킬 비교

| 스킬 | 이 정책이 갖지 못한 것 |
|------|------------------------|
| gstack `/review` | 신뢰도 1~10 점수 + 표시 규칙, 억제 목록(`DO NOT flag`) |
| mattpocock `code-review` (설치 1위) | Spec 축 — "컨벤션은 다 지켰는데 엉뚱한 걸 구현" 실패 모드 |
| 내장 `/code-review` | verify pass (CONFIRMED/PLAUSIBLE), `--fix`, `--comment` |

반대로 이 정책만 가진 강점은 리팩터링 가드레일, Structural Finding Contract,
교차 심문이다. **이번 변경은 강점을 건드리지 않는다.**

## 결정 사항

| 항목 | 결정 |
|------|------|
| 변경 파일 | `critical-review/agents/critical-code-reviewer.md` 1개 (+ 버전/문서) |
| 범위 | 리뷰 운영 품질 3종. 속도 개선은 범위 밖 |
| 강점 보존 | 리팩터링 가드레일·Structural Finding Contract·교차 심문 무수정 |
| 억제 목록 위치 | 레포 루트 `.critical-review-ignore.md` |
| 버전 | 0.1.0 → 0.2.0 |
| 브랜치 | `feat/critical-review-plugin` 계속 사용 (main 미머지 상태) |

### 속도를 범위 밖으로 두는 이유

측정된 9분 29초는 **공정한 베이스라인이 아니다.** 해당 실행에서 검증자가
SKILL.md 템플릿에 없는 조사 지시("호출부 4곳의 실제 사용처와 우회 경로를 확인하라")를
추가했고, 31회 툴 사용 중 상당수가 여기서 나왔다. 템플릿 원문은 "필요하면 조사하라"다.

정책 본문은 Codex판과 249줄 바이트 단위로 동일함을 확인했다. 즉 지연의 원인은
정책이 아니라 실행 조건(대상 크기, 조사 강제, 모델 속도)에 있다.

베이스라인 없이 워크플로를 재구성하는 것은 이 정책이 스스로 금지한
"증거 없는 구조 판단"에 해당한다. v0.2 적용 후 템플릿 원문으로 재측정하고
그 결과로 판단한다.

## 변경 1: 신뢰도 보정

**문제:** 현재 오탐 관리 장치는 Evidence Rules의 한 줄
(`Mark uncertain findings as risks instead of confirmed bugs`)뿐이다. 판단 주체가
finding을 생성한 그 추론이며, 등급도 표시 규칙도 없다.

**변경:** `Evidence Rules` 뒤에 `## Confidence Calibration` 섹션 신설.

- 9-10: 해당 코드를 읽고 실패 경로를 구체적으로 제시함
- 7-8: 읽은 코드에 대한 강한 패턴 일치
- 5-6: 개연성 있으나 미확인 — caveat를 붙여 보고
- 3-4: 의심 패턴뿐 — `## Findings`에서 제외하고 `## High-Risk Areas`로 강등
- 1-2: 추측 — Critical 급이 아니면 생략

finding 형식: `[Severity][Category](confidence: N/10) 제목`

기존 한 줄 규칙과 충돌하지 않는다. 그 규칙을 조작 가능한 등급으로 구체화한다.

## 변경 2: Spec 축

**문제:** Category 목록(`Correctness, Security, Reliability, Structure, Performance,
Test`)에 "요구된 것을 실제로 구현했는가"가 없다. 워크플로 2단계에
`Establish the intended behavior and review scope`가 있으나 finding 축이 아니다.

다른 모든 축을 통과하면서 실패할 수 있는 유일한 축이다. PR #40이 정확히 그 사례로,
가드 구현은 정상이나 설정이 XML 주석 안에 들어가 의도한 동작을 하지 않았다.

**변경:**
1. Category 목록에 `Spec` 추가
2. Evidence Rules에 규칙 추가 — 리뷰 대상에 명시된 의도(PR 본문, 이슈, 계획 파일,
   사용자 제공 요구사항)가 존재하면, 그것과 변경분을 대조한 결과를 **반드시 진술한다.**
   의도를 확인할 수 없으면 그 사실을 명시한다. 침묵하지 않는다.
3. 워크플로 2단계에 의도 출처 확보를 명시

## 변경 3: 억제 목록

**문제:** 레포마다 반복되는 오탐을 끌 방법이 없다. 같은 지적이 매 리뷰마다 재등장한다.

**변경:** 레포 루트 `.critical-review-ignore.md`를 리뷰 시작 시 확인하고, 존재하면
그 내용을 억제 규칙으로 적용한다. 파일이 없으면 아무것도 하지 않는다(신규 레포에서
동작 변화 없음).

- 억제된 항목은 `## Findings`에 올리지 않는다
- 억제 규칙이 **Critical 급 결함을 가리는 경우 억제를 무시하고 보고**한다.
  안전장치를 끄는 스위치가 되어서는 안 된다
- 무엇이 억제되었는지 리포트 말미에 1줄로 밝힌다(조용한 누락 방지)

파일 형식은 자유 서술 마크다운. 별도 스키마를 두지 않는다.

## 파일 변경 목록

| 파일 | 변경 |
|------|------|
| `critical-review/agents/critical-code-reviewer.md` | 섹션 3종 추가/수정 |
| `critical-review/.claude-plugin/plugin.json` | version 0.2.0 |
| `.claude-plugin/marketplace.json` | critical-review version 0.2.0 |
| `critical-review/README.md` | 신규 동작 3종 설명, 억제 파일 규약 |

## 완료 조건

- 정책의 리팩터링 가드레일·Structural Finding Contract·교차 심문 섹션이 무수정
- finding 출력에 confidence 점수가 나타난다
- 의도 출처가 있는 리뷰에서 Spec 축 진술이 나타난다
- `.critical-review-ignore.md`가 없는 레포에서 v0.1.0과 동작이 같다
- 버전 2곳(`plugin.json`, 루트 `marketplace.json`)이 `0.2.0` 으로 일치한다

## 범위 밖

- 속도/지연 개선 — 베이스라인 재측정 후 별도 결정
- verify pass, `--fix`, `--comment` 등 결과 배관
- `codex-review/` 동기화 (동결 유지)
