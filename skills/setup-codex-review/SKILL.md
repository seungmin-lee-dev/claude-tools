---
name: setup-codex-review
description: codex-review 플러그인의 Codex 쪽 의존성을 셋업한다. Codex CLI 설치·로그인 여부를 확인해 안내하고, 이 플러그인에 번들된 Codex 스킬(critical-code-review 등)을 사용자의 ~/.codex/skills/ 로 복사한다. codex-loop을 처음 쓰기 전에 팀원이 1회 실행. "codex 리뷰 셋업", "setup-codex-review", "코덱스 셋업" 요청 시 사용.
---

# setup-codex-review

codex-loop이 작동하려면 각 사용자 머신에 (1) Codex CLI 설치+로그인, (2) Codex 리뷰 스킬이
`~/.codex/skills/`에 있어야 한다. 이 스킬이 그 셋업을 자동화한다. 아래를 순서대로 수행한다.

## 1. Codex CLI 설치 확인
```bash
which codex || echo "NOT_INSTALLED"
```
- 설치돼 있으면 `codex --version` 출력.
- `NOT_INSTALLED`이면 사용자에게 설치를 안내하고 이후 단계는 보류:
  - 안내: "Codex CLI가 없습니다. 설치 후 다시 `/setup-codex-review`를 실행하세요."
  - 대표 설치법(사용자 환경에 맞게): `npm install -g @openai/codex` (Node 필요) 또는 Volta/브루 등 사내 표준 설치 경로.
  - 프로그램 설치는 자동 실행하지 말고 사용자가 직접 하도록 명령만 제시한다.

## 2. 로그인 확인
- 로그인은 브라우저 인증이라 자동화 불가. 사용자에게 안내:
  - "Codex 로그인이 안 돼 있으면 터미널에서 `! codex login` 을 한 번 실행하세요."
- (선택) 가벼운 비대화형 호출로 인증 여부를 탐지할 수 있으면 시도하고, 인증 에러가 나면 위 안내를 강조한다.
  실제 리뷰 호출을 여기서 돌려 쿼터를 쓰지는 말 것.

## 3. 번들된 Codex 스킬을 ~/.codex/skills/ 로 복사
이 스킬 디렉토리(런타임에 주어지는 "Base directory for this skill") 아래 `codex-skills/`에
번들된 스킬 폴더들을 사용자의 `~/.codex/skills/`로 복사한다. 이미 있으면 덮어쓰기 전 사용자에게 확인.

- macOS/Linux (bash):
```bash
SRC="<이 스킬 디렉토리>/codex-skills"
DEST="$HOME/.codex/skills"
mkdir -p "$DEST"
cp -r "$SRC/"* "$DEST/" 2>/dev/null && echo "copied Codex skills -> $DEST" || echo "번들된 codex-skills 없음(건너뜀)"
```
- Windows (PowerShell):
```powershell
$src = "<이 스킬 디렉토리>\codex-skills"
$dest = "$HOME\.codex\skills"
New-Item -ItemType Directory -Force $dest | Out-Null
if (Test-Path "$src\*") { Copy-Item "$src\*" $dest -Recurse -Force; "copied Codex skills -> $dest" } else { "번들된 codex-skills 없음(건너뜀)" }
```
- `codex-skills/`가 비어 있으면(사내 라이선스상 번들 안 한 경우) 복사를 건너뛰고, codex-loop은 그래도
  동작함(그 Codex 스킬 없이 기본 리뷰로)을 사용자에게 알린다.

## 4. 완료 보고
- Codex CLI 설치/로그인 상태, 복사된 Codex 스킬 목록, 남은 수동 작업(설치·로그인)을 요약한다.
- 마무리: "이제 아무 레포에서나 `/codex-loop`을 사용할 수 있습니다."
