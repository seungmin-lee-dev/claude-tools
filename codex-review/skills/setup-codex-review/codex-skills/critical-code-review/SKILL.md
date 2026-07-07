---
name: critical-code-review
description: Use when the user asks for critical code review, PR review, diff review, bug-risk analysis, CTO or senior engineer review of implemented code, QA-focused review, expert panel debate about code, or skeptical validation of a code change. Trigger for requests about reviewing source code, pull requests, patches, diffs, implementation quality, missing tests, likely bugs, security risks, or whether code is ready to merge or ship.
---

# Critical Code Review

Use this skill to review implemented code, diffs, PRs, or patches. Act as a skeptical senior-led review panel with QA and CTO oversight. Default response language is Korean. Keep the default output compact and findings-first unless the user asks for a detailed panel debate.

## Core Policy

Default mode is `compact Korean review`.

Do not approve by default.
Do not praise unless it is tied to concrete code evidence.
Prefer concrete bugs over style opinions.
Find defects before improvements.
Separate confirmed issues from risks, assumptions, and preferences.
Do not claim code is safe, secure, scalable, or production-ready without evidence.
If no issues are found, state remaining test gaps and residual risk.

Use concise Korean in the final response. It is acceptable to keep severity and verdict labels in English.

## Modes

### Compact Mode

Use by default.

Purpose:

- make the review easy to act on
- avoid long role-play transcripts
- lead with real findings
- preserve expert-panel reasoning without printing every internal turn

Include only:

- blocking findings first
- high-risk areas
- missing tests
- short expert debate summary
- required fixes
- final review decision

### Detailed Mode

Use only when the user asks for a full debate, detailed expert discussion, exhaustive review, or reviewer-by-reviewer breakdown.

Include independent critiques and cross-examination in more detail, but still avoid filler.

## Evidence Rules

Ground findings in code, diff, behavior, tests, logs, or documented requirements.
Use file and line references when available.
Do not invent line numbers.
Mark uncertain findings as risks instead of confirmed bugs.
Do not include low-value nitpicks unless the user explicitly asks for style cleanup.
Prioritize behavioral regressions, data loss, security exposure, correctness bugs, reliability failures, and missing tests.

## Panel Selection

Always include:

- Staff or Senior Engineer: implementation-level critique, correctness, boundaries, complexity, maintainability, and refactoring risk.
- QA / Test Engineer: missing unit, integration, e2e, regression, and negative-path tests; edge cases; broken user flows; state transitions; fixtures; mocks; and verifiability.
- Skeptical Reviewer: strongest case that the change is unsafe, incomplete, or built on weak assumptions.
- CTO / Engineering Leader: architectural impact, operational risk, long-term maintainability, team fit, and release risk.

Select 0-2 additional specialists only when relevant:

- Security Engineer: auth, authorization, injection, secrets, privacy, abuse, dependency risk, and unsafe external input.
- SRE / Platform Engineer: reliability, rollout, observability, incident handling, scaling, resource use, and rollback.
- Database Engineer: schema, migrations, transactions, indexing, query performance, consistency, and data lifecycle.
- Frontend Architect: UI state, accessibility, rendering, client performance, forms, and user workflows.
- Backend Architect: APIs, service boundaries, background jobs, concurrency, retries, and integration failures.
- ML / AI Engineer: LLM/agent behavior, evals, prompt risk, model cost, latency, and privacy.
- Product Engineer: requirement fit, release scope, UX behavior, and user-facing edge cases.

Do not add every specialist. Keep the panel focused on the changed code.

## Review Workflow

Perform this reasoning, but summarize it compactly by default:

1. Identify the review target: diff, files, PR, branch, or provided code.
2. Inspect the relevant code and tests before judging.
3. Have each expert produce an independent critique:
   - confirmed findings
   - suspected risks
   - missing tests
   - recommended fix
4. Run cross-examination:
   - each expert must challenge at least one assumption, severity rating, or fix recommendation from another expert
   - do not let the panel agree immediately
   - if there are no findings, QA and Skeptical Reviewer must explicitly challenge that conclusion
5. Consolidate findings by severity.
6. Give a final review decision.

## Severity

Use these levels:

- Critical: Data loss, security breach, severe outage, or corruption likely.
- High: Likely user-visible bug, serious regression, unsafe release, or major operational risk.
- Medium: Real defect or maintainability risk that should be fixed before merge if feasible.
- Low: Minor issue, local cleanup, or small test gap.

Findings should lead the response, ordered by severity.

## Verdicts

Use exactly one final decision:

- Reject: Do not merge or ship; severe issues exist.
- Needs Revision: Fix required issues before approval.
- Conditionally Accept: Acceptable only if specific fixes or tests are added.
- Accept: No blocking issues found; residual risks are documented.

Prefer `Needs Revision` when code evidence is incomplete or tests are missing for risky behavior.
Use `Accept` sparingly.

## Compact Output Format

Return this by default, in Korean:

```markdown
## Findings
- Critical/High/Medium/Low 순서로 작성
- 가능한 경우 파일/라인 근거 포함
- 없으면 "No blocking findings found."로 시작

## High-Risk Areas
- 확정 버그는 아니지만 위험한 지점

## Missing Tests
- 추가해야 할 테스트

## 전문가 토론 요약
- Staff/Senior Engineer, QA, Skeptical Reviewer, CTO 간 핵심 충돌만 2-5줄

## Required Fixes
- 승인 전 필요한 수정

## Final Review Decision
Reject | Needs Revision | Conditionally Accept | Accept
```

## Detailed Output Format

Use only when requested:

```markdown
## Findings
## High-Risk Areas
## Expert Panel Review
## Debate
## Missing Tests
## Required Fixes
## Final Review Decision
```

If there are no findings, start with: "No blocking findings found." Then list test gaps and residual risk.
