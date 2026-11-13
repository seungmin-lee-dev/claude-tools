---
name: ego-qa
description: ego-browser(ego lite)로 로컬 웹 앱을 사람처럼 눌러 보며 QA 한다. 내 작업 화면 확인, 남의 PR 을 받아 실제로 되는지 시험, 버그 재현, PR 용 수정 전·후 캡처, 콘솔 오류·실패한 요청 수집에 쓴다. "QA 해줘", "화면 되는지 봐줘", "PR 123 실제로 되는지 테스트해줘", "전후 캡처 찍어줘", "이 버그 재현해줘" 같은 요청에 쓴다. 프로젝트별 주소·계정·규칙은 그 프로젝트 쪽 스킬이나 AGENTS.md 에서 읽는다.
---

# ego QA

ego-browser 로 로컬 화면을 직접 조작하고, 콘솔 오류·실패한 요청까지 모아 통과/실패를 보고한다.

## 0. 시작 전에

1. `ego-browser skill` 을 실행해 출력(사용 설명서)을 따른다. ego 사용법의 원본은 그 출력이다. 이 스킬은 QA 순서만 담는다.
2. 명령을 못 찾으면 ego lite 설치 여부를 확인한다. 설치 직후라면 새 터미널에서 다시 시도한다(PATH 는 새로 연 프로세스에만 적용된다).
3. Bash 계열은 `ego-browser nodejs <<'EOF' … EOF`, PowerShell 은 `.mjs` 파일 방식으로 실행한다(설명서 참고).
4. **프로젝트 값 찾기**: 앱 주소·포트, 시험 계정, 서버 띄우는 방법, 금지 사항을 다음 순서로 찾는다.
   프로젝트 전용 QA 스킬 → 프로젝트 `AGENTS.md`·`CLAUDE.md` → 개발 서버 실행 스크립트·`package.json`. 계정을 못 찾으면 사용자에게 묻는다.

## 1. 기본 QA 순서

1. 시나리오를 3~7줄로 먼저 적는다(어느 화면 → 무엇을 누르면 → 무엇이 보여야 함).
2. 서버가 켜졌는지 먼저 본다: `curl -s -o /dev/null -w "%{http_code}" <주소>`. 꺼져 있으면 사용자에게 알리거나 허락받고 띄운다.
3. 목표 하나에 작업 공간(TaskSpace) 하나. 앱 여러 개를 동시에 볼 때, 여러 이슈가 묶인 PR(4절)일 때, 사용자가 나눠 달라고 할 때만 공간을 더 연다.
4. 로그인 직후 오류 수집을 켠다(2절). 그다음 시나리오를 진행한다.
5. 관찰은 스냅샷(글로 바꾼 화면)이 기본이다. 스크린샷은 눈으로 확인해야 할 때와 증거·PR 캡처에만 찍는다. 이미지는 토큰이 많이 든다.
6. 여러 동작을 한 스크립트에 묶고, 마지막에 스냅샷 하나만 출력한다.
7. 끝나면 `task.finish({ keep: [] })`. 사용자가 결과 화면을 남겨 달라고 했을 때만 `keep` 에 넣는다.

## 2. 콘솔 오류·실패한 요청 모으기

ego 는 기본으로 콘솔을 숨긴다(사이트가 자동 조종을 알아채지 못하게). **내 로컬 앱 QA 에서만 켠다.** 외부 사이트에서는 켜지 않는다.

켜기 (공간을 연 첫 스크립트에서 한 번):

```js
await page.cdp("egoAutomated.disable");
await page.cdp("Runtime.enable");
await page.cdp("Network.enable");
```

시나리오를 진행한 스크립트 끝에서 요약 (이벤트는 한 번 읽으면 비워진다):

```js
const issues = [];
for (const e of await page.events()) {
  const p = e.params;
  if (e.method === "Runtime.consoleAPICalled" && p.type === "error")
    issues.push(`console.error: ${p.args.map((a) => a.value ?? a.description ?? "").join(" ").slice(0, 300)}`);
  else if (e.method === "Runtime.exceptionThrown")
    issues.push(`exception: ${(p.exceptionDetails.exception?.description ?? p.exceptionDetails.text).slice(0, 300)}`);
  else if (e.method === "Network.responseReceived" && p.response.status >= 400 && !p.response.url.startsWith("chrome://"))
    issues.push(`HTTP ${p.response.status} ${p.response.url}`);
  else if (e.method === "Network.loadingFailed" && !p.canceled)
    issues.push(`network failed: ${p.errorText} (${p.type})`);
}
console.log({ url: await page.url(), issues });
```

- `waitForURL()` 은 문자열·정규식으로 준다. 함수(predicate)는 받지 않는다.
- 응답 본문이 필요하면 `page.cdp("Network.getResponseBody", { requestId })`.

## 3. 남의 PR 을 받아서 실제로 되는지 시험

1. **PR 파악**: `gh pr view <번호> --json title,body,headRefName,files,closingIssuesReferences`. 본문의 테스트 방법·연결 이슈로 시나리오를 만든다. 바뀐 파일로 어느 앱·화면인지 정한다.
2. **위험한 변경 확인**: DB 마이그레이션·공용 데이터 변경이 있으면 공용 개발 DB 에 적용하지 않는다. 사용자에게 알리고, 프로젝트가 정한 시험 DB 방식을 따르거나 그 변경 없이 볼 수 있는 화면만 본다.
3. **워크트리 만들기**: 지금 작업을 건드리지 않게 따로 받는다.
   ```bash
   git fetch origin <headRefName>
   git worktree add ../<레포>-pr<번호> origin/<headRefName>
   ```
   git 에 없는 로컬 설정 파일(`.env*` 등)이 필요하면 원래 폴더에서 복사한다. 의존성을 설치한다.
4. **서버 띄우기**: 이미 켜진 내 서버와 포트가 겹치지 않게 한다. 실행 스크립트가 PID·상태 파일을 공용 위치에 쓰면 그 위치도 나눈다(안 나누면 내 서버 기록을 덮는다).
   **localhost 쿠키는 포트를 가리지 않는다.** 같은 앱을 포트만 바꿔 두 벌 띄우면 로그인 쿠키가 서로 덮인다. 그래서 PR 여러 개를 같은 ego 프로필로 동시에 시험하지 않는다(꼭 해야 하면 ego 프로필을 나눈다 — `help("profiles")`).
5. **시험**: 1·2절 순서로 진행한다. PR 이 고쳤다는 것 + 그 주변(같은 화면의 다른 버튼, 새로고침 뒤 유지, 언어 전환, 좁은 화면)을 본다.
6. **증거**: 실패한 단계와 핵심 결과만 스크린샷을 찍는다. 절대 경로로 저장한다(Windows 도 `/` 를 쓴다).
7. **보고**: 사용자에게 아래 표로 알린다. **PR 에 코멘트를 다는 건 사용자가 허락했을 때만** 한다.

   | 시나리오 | 결과 | 근거 |
   |---|---|---|
   | ① 로그인 → 목록 → 수정 저장 | 통과 / 실패 | 스냅샷 요약·오류 문구·캡처 |

   콘솔 오류·HTTP 4xx/5xx 는 통과한 시나리오에서 나와도 따로 적는다.
8. **정리**: 띄운 서버 끄기 → `task.finish({ keep: [] })` → `git worktree remove ../<레포>-pr<번호>`. 다시 볼 계획이면 사용자에게 묻고 남긴다.

## 4. 여러 이슈가 묶인 PR (에픽) — 이슈별 병렬 QA

에픽 PR 처럼 하위 이슈가 여러 개면, **서버는 한 벌만 띄우고 이슈마다 작업 공간을 하나씩 열어 동시에** 시험한다.
이 경우는 사용자가 따로 말하지 않아도 공간을 여러 개 연다.

1. **이슈 목록**: PR 본문의 이슈별 절, `closingIssuesReferences`, 에픽의 하위 이슈(`gh api repos/<owner>/<repo>/issues/<에픽번호>/sub_issues --jq '.[] | {number,title,state}'`)로 모은다. 이슈마다 시나리오를 3~7줄로 만든다.
2. **서버 한 벌**: 3절 3~4번대로 워크트리·서버를 **한 번만** 준비한다. 로그인도 앱마다 한 번 해 둔다(공간끼리 브라우저 로그인을 같이 쓴다).
3. **나누기**: 시나리오를 두 묶음으로 가른다.
   - 동시에 돌려도 되는 것: 조회·화면 확인, 서로 다른 데이터를 만드는 입력.
   - 순서대로 돌릴 것: 로그아웃·계정 전환, 다른 화면에 영향을 주는 설정 변경, 같은 데이터를 고치는 것.
4. **병렬 실행**: 동시에 돌릴 묶음은 이슈 하나당 하위 에이전트 하나(Claude 는 Agent 도구, 백그라운드)에 맡긴다. 각 에이전트는 자기 공간을 `taskSpace("PR<번호> #<이슈> <요약>")` 로 열고, 시나리오·주소·계정·2절 오류 수집 코드를 프롬프트로 받는다. **한 번에 3~4개까지.** 하위 에이전트를 못 쓰는 환경(Codex 등)에서는 이슈별 스크립트를 백그라운드로 동시에 실행한다.
5. **순차 실행**: 병렬 묶음이 끝난 뒤 순서대로 돌린다.
6. **보고**: 이슈별로 묶어서 하나의 표로 낸다.

   | 이슈 | 시나리오 | 결과 | 근거 |
   |---|---|---|---|
   | #101 결제창 카드만 | ① 결제창 열기 → 카드 칸만 보임 | 통과 | 스냅샷 요약 |

7. **정리**: 모든 공간을 `finish` 한 뒤 서버를 한 번 끄고 워크트리를 지운다.

## 5. PR 용 수정 전·후 캡처

- 화면 코드를 고치기 **전에** 전 화면을 먼저 찍는다. 나중에는 못 찍는다. 기본 브랜치를 띄운 다른 워크트리의 서버로 찍어도 된다.
- 같은 공간·같은 창 크기에서 `page.screenshot({ path, fullPage: false })` 로 찍어 크기를 맞춘다.
- 상태를 보여 주려고 공용 DB 데이터를 바꾸지 않는다. 필요하면 응답만 바꿔서 찍고 PR 에 그렇게 적는다.

## 하지 말 것

- 로그인이 막히면 비밀번호를 추측하거나 계정을 새로 만들지 않는다. `task.handOff()` 로 사용자에게 넘긴다.
- 막힌 공간을 버리고 새 공간을 열지 않는다. 멈추고 사용자에게 묻는다.
- 운영(실서비스) 주소에서 QA 하지 않는다. 확인만 필요하면 먼저 묻는다.
