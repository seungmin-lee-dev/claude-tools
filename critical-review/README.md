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
                          각 finding에 confidence 1~10 표기
## High-Risk Areas
## Missing Tests
## 전문가 토론 요약
## Required Fixes
## Follow-up Improvements
## Final Review Decision   (Reject | Needs Revision | Conditionally Accept | Accept)
## 억제                    (무엇이 억제되었는지 1줄)
```

## 신뢰도 점수

모든 finding에 `(confidence: N/10)` 이 붙는다. 코드를 실제로 읽고 실패 경로를
제시했으면 9~10, 조사하지 않았으면 6을 넘을 수 없다.

**5 미만은 `## Findings` 에 오르지 않고 `## High-Risk Areas` 로 내려간다.**
오탐이 확정 결함처럼 보이는 것을 막기 위한 장치다.

## Spec 축

리뷰 대상에 명시된 의도(PR 본문·연결된 이슈·계획 문서·직접 준 요구사항)가 있으면
변경분과 대조해 `Spec` 카테고리로 진술한다. 의도 출처가 없으면 없다고 밝힌다.

다른 축을 전부 통과하면서도 실패할 수 있는 유일한 축이다 — 컨벤션을 다 지키면서
엉뚱한 것을 구현한 코드가 여기서 걸린다.

## 억제 목록

레포 루트에 `.critical-review-ignore.md` 가 있으면 그 내용을 억제 규칙으로 읽는다.
파일이 없으면 아무 동작 변화가 없다.

- 형식 자유. 끄고 싶은 지적을 서술하면 된다
- **Critical 급에는 억제가 적용되지 않는다.** 데이터 손실·보안 침해·심각한 장애를
  가리는 규칙은 무시하고 보고한다
- 알려진 허용 패턴만 침묵시킨다. 그것을 닮은 새 결함은 침묵시키지 않는다
- 무엇이 억제되었는지 리포트 `## 억제` 에 남는다 — 조용한 누락은 없다

## 비용

리뷰 추론이 클로드 쪽에서 돈다. 서브에이전트 격리는 **메인 대화 컨텍스트**를 보호하는 것이지
총 토큰 사용량을 줄이지는 않는다. 큰 diff는 범위를 좁혀 여러 번 돌리는 편이 낫다.

## 리뷰 정책 수정

정책 정본은 [`agents/critical-code-reviewer.md`](agents/critical-code-reviewer.md) 다.
같은 정책의 Codex용 사본이 `codex-review` 플러그인에 있으나 현재 **동결** 상태이며 동기화하지 않는다.

## 업데이트

push 후 팀원은 `/plugin`에서 업데이트. `plugin.json`과 루트 `marketplace.json`의 `version`을 함께 올린다.
