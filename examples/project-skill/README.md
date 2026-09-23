# 프로젝트 스킬 예시 — `/cutflow`

`.claude/skills/cutflow/SKILL.md`는 [CutFlow 저장소](https://github.com/wintercat8/test1)에 들어 있는 **프로젝트 스킬** 원본입니다 (플러그인 설치 없이, 그 저장소를 연 사람에게 바로 보이는 방식).

여기에 둔 이유는 플러그인을 설치하지 않고도 스킬 내용을 읽어볼 수 있게 하기 위해서입니다.

## 두 방식의 차이

| | 프로젝트 스킬 (이 폴더) | 플러그인 스킬 ([plugins/cutflow-kit](../../plugins/cutflow-kit)) |
|---|---|---|
| 위치 | 앱 저장소 안 `.claude/skills/cutflow/` | 이 마켓플레이스 |
| 호출 | `/cutflow` | `/cutflow-kit:cutflow` |
| 적용 범위 | 그 저장소를 열었을 때만 | 설치하면 모든 프로젝트에서 |
| 차이점 | 현재 폴더가 CutFlow라고 가정 | **0단계**가 있어 CutFlow 코드를 찾거나 받아옴 |

내용은 그 0단계를 빼면 같습니다.

## 내 프로젝트에 프로젝트 스킬로 넣으려면

`.claude/skills/cutflow/SKILL.md`를 내 저장소의 같은 경로에 복사하면 됩니다. Claude Code를 다시 열면 `/cutflow`로 쓸 수 있습니다.

```
내-저장소/
└─ .claude/
   └─ skills/
      └─ cutflow/
         └─ SKILL.md
```

`SKILL.md`는 첫 줄이 `---`(YAML 프론트매터)로 시작해야 스킬로 인식됩니다. 앞에 다른 줄을 넣지 마세요.

## 직접 스킬을 만들 때 참고할 점

이 파일은 아래 요소를 담고 있어 예시로 쓰기 좋습니다.

- `description`에 사용자가 실제로 쓸 법한 말("쇼츠 뽑아줘")을 넣어, Claude가 알아서 스킬을 고를 수 있게 함
- `argument-hint`로 기대하는 인수 표시
- 요금이 드는 작업(AI 호출)은 **실행 전에 사용자에게 알리도록** 명시
- 사용자 파일을 확인 없이 지우지 않기, API 키를 출력하지 않기 같은 **하지 말 것** 목록
