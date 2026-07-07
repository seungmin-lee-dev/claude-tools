# claude-tools

팀 내부 Claude Code 도구 모음 (플러그인 마켓플레이스).
한 레포에 여러 플러그인을 담고, 팀원은 이 마켓플레이스를 한 번 등록한 뒤 필요한 플러그인을 설치한다.

## 마켓플레이스 등록 (팀원 1회)
```
/plugin marketplace add seungmin-lee-dev/claude-tools
```

## 플러그인 목록

### codex-review
Codex CLI 기반 리뷰·문서 오프로드 워크플로. 코드는 클로드, 리뷰·문서는 Codex → 클로드 토큰 절감.
```
/plugin install codex-review@claude-tools
```
- 스킬: `/codex-review:codex-loop`, `/codex-review:setup-codex-review`
- 자세한 내용: [`codex-review/README.md`](codex-review/README.md)

## 새 도구(플러그인) 추가하는 법
1. `<플러그인명>/.claude-plugin/plugin.json` + `<플러그인명>/skills/...` 로 하위 폴더 추가
2. 루트 `.claude-plugin/marketplace.json`의 `plugins[]`에 `{ "name": ..., "source": "./<플러그인명>", ... }` 등록
3. 버전 올리고 push → 팀원은 `/plugin`에서 업데이트

## 구조
```
.claude-plugin/marketplace.json   # 이 레포 = 마켓플레이스
codex-review/                     # 플러그인 (하위 폴더)
  .claude-plugin/plugin.json
  skills/
    codex-loop/SKILL.md
    setup-codex-review/SKILL.md
```
