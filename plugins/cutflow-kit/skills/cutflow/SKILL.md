---
name: cutflow
description: CutFlow로 유튜브 영상을 자동 편집한다 — 무음 구간 제거, 쇼츠(9:16) 추출, 제목·설명·태그·챕터 메타데이터, 썸네일 후보, 자막, (API 키가 있으면) AI 자막 교정·번역·블로그·홍보 문구. "이 영상 편집해줘", "쇼츠 뽑아줘", "inbox 처리해줘", "메타데이터 만들어줘" 같은 요청에 사용.
argument-hint: "[영상 경로… | inbox] [프리셋: default|shorts|podcast|global]"
---

# CutFlow 영상 자동 편집

요청: $ARGUMENTS

이 저장소의 `app/cli.py`로 영상을 처리하고, 결과를 사용자에게 요약해 준다. 명령은 항상 **저장소 루트**에서 실행한다.

## 0. CutFlow 코드 찾기 (플러그인으로 설치된 경우)

이 스킬은 어느 프로젝트에서나 불릴 수 있다. 먼저 CutFlow 코드가 어디 있는지 확인한다.

1. 현재 작업 폴더에 `app/cli.py`와 `workflow.json`이 있으면 여기가 CutFlow다. 1단계로 넘어간다.
2. 없으면 사용자에게 CutFlow 폴더 경로를 묻는다. 모르면 받아올지 묻고, 동의하면 사용자가 정한 위치에 받는다.
   - `git clone https://github.com/wintercat8/test1.git cutflow` (비공개 저장소라 접근 권한이 필요하다)
   - `pip install -r requirements.txt`
3. 이후 명령은 모두 그 CutFlow 폴더를 작업 디렉터리로 해서 실행한다. 영상 경로는 절대 경로로 넘긴다.

## 1. 무엇을 처리할지 정하기

- 인수나 대화에 **영상 경로**가 있으면 → `process` (원본은 그대로 둔다)
- `inbox`라고 하거나 경로가 없으면 → `inbox/` 폴더에 있는 영상을 `watch --once`로 처리 (처리한 원본은 `inbox/done/`으로 옮겨짐)
- 경로도 없고 `inbox/`도 비어 있으면, 처리할 영상 경로를 물어본다.

## 2. 프리셋 / 기능 고르기

먼저 `python -m app.cli presets`로 프리셋과 API 키 상태를 확인한다. 요청에 맞게 고른다.

| 요청 예시 | 선택 |
|---|---|
| "편집해줘", 특별한 말 없음 | 프리셋 `default` |
| "쇼츠만", "쇼츠 뽑아줘" | 프리셋 `shorts` |
| 팟캐스트·토크·강의 | 프리셋 `podcast` |
| 해외·영어 자막 | 프리셋 `global` |
| 특정 기능만 ("무음만 잘라줘") | `--actions silence` 처럼 직접 지정 |

`--actions`에 쓸 수 있는 값은 다음과 같다.
- 편집 기능: `silence`, `shorts`, `metadata`, `thumbnails`, `transcribe`
- AI 기능: `ai_metadata`, `ai_shorts`, `ai_fix_subs`, `ai_translate`, `ai_titles`, `ai_blog`, `ai_social`

**AI 기능은 Claude API를 호출해 요금이 나온다.**
- API 키가 있고 선택한 프리셋에 AI 기능이 포함돼 있으면, 실행 전에 "AI 기능(…)도 실행되어 API 사용료가 발생합니다"라고 알린다.
- 사용자가 AI 기능을 명시적으로 요청했거나 이미 동의한 경우에만 그대로 진행한다.
- 키가 없으면 AI 기능은 자동으로 건너뛰므로 따로 알릴 필요는 없다.

## 3. 실행

```bash
python -m app.cli process "<영상 경로>" --preset <프리셋>
python -m app.cli process "<영상 경로>" --actions silence,metadata
python -m app.cli watch --once --preset <프리셋>
```

- 경로에 공백이나 한글이 있으면 반드시 따옴표로 감싼다.
- 긴 영상은 수 분 이상 걸린다 (특히 음성 인식). 10분이 넘는 영상은 백그라운드로 실행하고, 끝나면 결과를 확인한다.
- 패키지가 없다는 오류가 나면 `pip install -r requirements.txt`를 제안한다 (설치는 사용자 확인 후).
- 음성 인식 모델은 첫 실행 때 내려받는다 (small ≈ 480MB). 처음이면 미리 알린다.

## 4. 결과 보고

마지막 줄에 출력되는 `→ outbox/<이름_날짜>/` 폴더의 `요약.txt`를 읽고 아래를 짧게 정리한다.

- 무음 제거: 줄어든 시간과 비율, 컷 편집본 파일
- 쇼츠: 개수와 각 구간 (시작~끝, 제목)
- 제목 후보 상위 3개, 태그 개수, 챕터 유무
- AI 결과 (있다면): 자막 교정 줄 수, 블로그 글·홍보 문구 파일
- 결과 폴더 경로

실패하면 (`✖ 실패`, 또는 `inbox/failed/*_오류.txt`가 생긴 경우) 오류 내용을 그대로 보여주고, 원인(손상된 파일, 오디오 없음, 키 오류 등)과 해결 방법을 제안한다.

## 5. 추가 작업

- 결과를 화면에서 보거나 기능을 더 실행하려면 웹앱을 안내한다: `python run.py` → http://127.0.0.1:8765 → 작업 기록. CLI로 처리한 작업도 여기에 나온다.
- 옵션을 바꿔 다시 실행할 때 (예: 쇼츠 개수, 무음 기준)는 `workflow.json` 프리셋을 고치기보다 웹앱의 기능 실행 버튼을 쓰는 편이 간단하다. 프리셋을 영구히 바꾸고 싶다고 하면 그때 `workflow.json`을 수정한다.

## 하지 말 것

- `workspace/`, `inbox/done/`, `outbox/`의 파일을 사용자 확인 없이 지우지 않는다 (사용자의 원본과 결과물).
- API 키를 출력하거나 파일에 쓰지 않는다. 키는 웹앱 설정 화면이나 `.env`로만 넣게 안내한다.
