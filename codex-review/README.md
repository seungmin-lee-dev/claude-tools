# codex-review (Claude Code 플러그인)

코드는 **클로드**가 짜고, 리뷰·문서는 **Codex CLI**에 오프로드하는 리뷰 워크플로.
무거운 리뷰·문서 추론이 Codex(OpenAI) 쪽에서 돌아 **클로드 토큰을 절감**한다.
플러그인으로 설치하면 **모든 레포/경로에서** 사용 가능.

## 포함 스킬 (설치 후 네임스페이스로 호출)
- **`/codex-review:codex-loop`** — 현재 diff(또는 지시한 기능 구현분)를 Codex `critical-code-review`로
  리뷰 → 클로드가 수정 → 재리뷰를 clean 또는 최대 N회까지 반복 → Codex가 문서 갱신. 커밋은 안 함.
  - 인자 없음: 현재 uncommitted diff / `<작업 지시>`: 클로드가 먼저 구현
  - 플래그: `--max-rounds N`(기본 3), `--no-docs`, `--design-review`
- **`/codex-review:setup-codex-review`** — Codex 쪽 의존성 셋업(1회): Codex CLI 설치/로그인 확인 안내 +
  번들 Codex 스킬을 `~/.codex/skills/`로 복사.

> 자연어로 "코덱스로 리뷰하고 고쳐줘"처럼 요청하면 네임스페이스 없이도 스킬이 트리거된다.

## 팀원 설치 (1회)
```
# 1) 마켓플레이스 등록
/plugin marketplace add seungmin-lee-dev/claude-tools

# 2) 플러그인 설치
/plugin install codex-review@claude-tools

# 3) Codex 쪽 셋업 (아무 레포에서나 1회)
/codex-review:setup-codex-review
```
설치 후 **아무 경로/레포에서나** `/codex-review:codex-loop` 사용 가능.

## 전제 (각 팀원 머신 1회)
- **Codex CLI 설치 + `codex login`** — 프로그램이라 플러그인에 담기지 않음. `setup-codex-review`가
  설치/로그인 여부를 확인해 안내한다. (대표 설치: `npm install -g @openai/codex`)
- (선택) `skills/setup-codex-review/codex-skills/`에 번들된 Codex 스킬 — 셋업 시 `~/.codex/skills/`로 복사.

## 비용
- 리뷰·문서 추론 = Codex(OpenAI) 쿼터 소모, 클로드 토큰은 최소.
- 무한 루프 방지를 위해 `--max-rounds`(기본 3)로 제한.

## 업데이트
플러그인 레포에 push 후 팀원은 `/plugin`에서 업데이트. `plugin.json`과 루트 `marketplace.json`의
`version`을 함께 올린다.
