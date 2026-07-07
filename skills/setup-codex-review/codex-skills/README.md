# 번들 Codex 스킬 (codex-skills/)

여기에 팀에 배포할 **Codex 스킬 폴더**를 넣으면, 팀원이 `/setup-codex-review` 실행 시
각자의 `~/.codex/skills/`로 자동 복사된다.

예:
```
codex-skills/
  critical-code-review/
    SKILL.md
  critical-design-review/
    SKILL.md
```

주의:
- 여기에 넣는 Codex 스킬은 **재배포 권한이 있는(사내 자체 제작 등)** 것만 넣을 것.
  제3자 배포본을 사내 공유하려면 출처·라이선스를 확인한다.
- 이 폴더가 비어 있어도 `codex-loop`은 동작한다(그 스킬 없이 Codex 기본 리뷰로).
