# ego-qa

ego-browser(ego lite)로 로컬 웹 앱을 사람처럼 눌러 보며 QA 하는 스킬.

- 설치: `/plugin install ego-qa@claude-tools`
- 스킬: `/ego-qa:ego-qa` (자연어 "QA 해줘", "PR 123 실제로 되는지 테스트해줘"도 트리거)
- 필요: [ego lite](https://github.com/citrolabs/ego-lite) 설치 + 첫 실행 설정 완료 (`ego-browser` 명령이 잡혀야 함)

## 하는 일
- 시나리오를 세우고 화면을 직접 조작해 통과/실패를 표로 보고
- 콘솔 오류·예외·HTTP 4xx/5xx·실패한 요청을 모아서 함께 보고
- 남의 PR 을 별도 워크트리·별도 포트로 띄워 시험하고 정리
- PR 용 수정 전·후 캡처

## 프로젝트 값
앱 주소·시험 계정·서버 띄우는 법은 이 스킬에 없다. 프로젝트 전용 QA 스킬이나 `AGENTS.md` 에서 읽는다.
회사 내부 값(계정·도메인 등)은 공개 저장소가 아니라 비공개 스킬 저장소에 둔다.
