# wintercat8 Claude Code 스킬 모음

Claude Code 플러그인 마켓플레이스입니다. 설치하면 어느 프로젝트에서든 아래 스킬을 쓸 수 있습니다.

## 설치

Claude Code에서 실행합니다.

```
/plugin marketplace add wintercat8/claude-skills
/plugin install cutflow-kit@wintercat8
```

업데이트는 `/plugin marketplace update wintercat8`로 받습니다. 새 버전은 `plugin.json`의 `version`을 올려야 배포됩니다.

## cutflow-kit

| 스킬 | 호출 | 하는 일 |
|---|---|---|
| `cutflow` | `/cutflow-kit:cutflow [영상 경로 \| inbox] [프리셋]` 또는 "이 영상 쇼츠 뽑아줘" | [CutFlow](https://github.com/wintercat8/test1) 앱으로 영상 자동 편집: 무음 제거, 쇼츠, 메타데이터, 썸네일, (키가 있으면) AI 기능. 결과를 요약해 줍니다. |
| `build-local-app` | `/cutflow-kit:build-local-app [앱 설명] [레퍼런스 URL]` | CutFlow를 만든 과정 그대로, 디자인 레퍼런스를 반영한 로컬 자동화 웹앱을 처음부터 만듭니다. FastAPI + 바닐라 JS, 기능별 버튼, API 키로 여는 AI 기능, 폴더 감시 CLI, pytest + GitHub Actions까지 포함합니다. |

- `build-local-app`은 큰 작업이라 **직접 호출할 때만** 실행되도록 설정했습니다 (`disable-model-invocation`).
- `cutflow` 스킬은 CutFlow 코드가 필요합니다. 현재 폴더가 CutFlow가 아니면 경로를 묻거나, 동의를 받은 뒤 저장소를 받아옵니다.

## 구조

```
.claude-plugin/marketplace.json
plugins/cutflow-kit/
  .claude-plugin/plugin.json
  skills/
    cutflow/SKILL.md
    build-local-app/
      SKILL.md          단계별 절차와 완료 기준
      architecture.md   Job 액션 파이프라인, 서버 API, 보안 규칙
      design.md         레퍼런스 → 디자인 토큰 → 구성 요소
      ai-layer.md       API 키로 여는 AI 기능 (Claude)
      testing-ci.md     격리 테스트, 입력 생성, CI 템플릿
```

## 주의

- 스킬에는 API 키 같은 비밀 값을 넣지 않습니다.
- 스킬이 실행하는 명령은 사용자 PC에서 돌아갑니다. 파일 삭제, 비용이 드는 API 호출, 시작프로그램 등록은 스킬 안에서 사용자 확인을 받게 되어 있습니다.
