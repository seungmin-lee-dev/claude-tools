---
name: codex-loop
description: 코드는 클로드가 짜고, 리뷰와 문서는 Codex CLI에 오프로드하는 반복 개선 루프. Codex의 critical-code-review 스킬로 현재 diff(또는 클로드가 방금 구현한 코드)를 리뷰 → 클로드가 지적을 반영해 코드 수정 → Codex 재리뷰를 clean 또는 최대 N회까지 반복 → 마지막에 Codex가 문서 갱신. 무거운 리뷰·문서 추론을 Codex(OpenAI)로 넘겨 클로드 토큰을 절감한다. "codex 루프", "코덱스로 리뷰하고 고쳐줘", "codex-loop", "리뷰 개선 반복" 요청 시 사용.
---

# codex-loop

코드=클로드, 리뷰·문서=Codex. 클로드가 Codex CLI(`codex exec`)를 비대화형으로 몰아
**리뷰 → (클로드가) 수정 → 재리뷰**를 반복하고, 끝에 문서를 Codex가 갱신한다.
무거운 리뷰·문서 작업이 Codex(OpenAI) 쪽에서 돌아 클로드 토큰을 아낀다.

MCP 불필요. 아래 절차를 순서대로 따른다.

## 0. 인자 파싱
- 인자 없음 → **현재 uncommitted diff**를 대상으로 시작.
- `<작업 지시>`(예: "이메일 중복검사 추가") → 클로드가 **먼저 구현**한 뒤 루프.
- 플래그: `--max-rounds N`(기본 3), `--no-docs`(문서 단계 생략), `--design-review`(critical-design-review 관점 추가).

## 1. 전제 확인
- git 레포인지 확인. Codex CLI 존재 확인: `which codex` (없으면 사용자에게 알림).
- 이후 codex 호출이 인증 에러(로그인 필요)로 실패하면, 사용자에게 터미널에서 `! codex login` 실행을 안내하고 중단.

## 2. (모드 ②) 구현
작업 지시가 주어졌으면 클로드가 그 기능을 구현한다(코드 작성). 지시가 없으면 건너뛴다.

## 3. 대상 diff 확보
`git diff`(uncommitted)를 대상 diff로 둔다. 비어 있으면 "리뷰할 변경이 없음"을 알리고 종료.

## 4. 리뷰 루프 (최대 `--max-rounds`, 기본 3)
아래를 반복한다:

1. **Codex 리뷰 (read-only)** — 클로드가 Bash로 실행:
   ```bash
   git diff | codex exec -s read-only "critical code review: 아래 <stdin>의 diff를 critical-code-review 관점(QA+CTO, 회의적 시니어 패널)으로 리뷰하라. 확정 버그를 우선, findings-first로 한국어로. 마지막 줄에 반드시 'VERDICT: CLEAN' 또는 'VERDICT: ISSUES' 한 줄을 출력하라."
   ```
   - diff가 매우 크면(대략 6만 토큰 초과) 파일 단위로 나눠 호출하거나 범위 축소를 제안.
2. **판정 읽기** — Codex 출력의 마지막 `VERDICT:` 줄과 findings를 읽는다.
3. **분기**:
   - `VERDICT: CLEAN` → 루프 종료(5단계로).
   - `VERDICT: ISSUES` → 클로드가 findings 중 실질 이슈를 **코드로 직접 수정**한다. 수정 후 다음 라운드로.
4. **라운드 카운트** — `--max-rounds` 도달 시, 남은 미해결 findings를 보고에 남기고 루프 종료.

각 `codex exec`는 비대화형이라 승인 프롬프트로 멈추지 않는다. 리뷰 단계는 항상 `-s read-only`(Codex가 코드를 못 건드림).

## 5. 문서 갱신 (`--no-docs`면 생략)
클로드가 Bash로 실행 — Codex가 문서를 직접 편집(workspace-write):
```bash
codex exec -s workspace-write "이번 변경(git diff 참고)에 맞춰 관련 문서를 갱신하라: README/CHANGELOG/설정 문서/공개 API 주석 등 실제로 영향받는 것만. 코드 로직은 바꾸지 말고 문서·주석만 수정하라. 한국어."
```
- `--design-review`면 프롬프트에 "critical-design-review 관점(구조·인터페이스·문서 일관성)"을 추가.
- 실행 후 클로드가 `git diff`로 문서 변경을 확인한다(코드 파일이 바뀌었으면 사용자에게 알림).

## 6. 보고
다음을 요약한다:
- 리뷰 라운드 수, 각 라운드 핵심 findings, 클로드가 고친 내역(파일별).
- 최종 Codex 판정(CLEAN / max-rounds 도달 시 미해결 목록).
- 문서 변경 요약.
- **커밋은 하지 않는다**(사용자 몫). 비용 참고: 리뷰·문서 추론은 Codex(OpenAI) 쿼터 소모, 클로드 토큰은 최소.
