# 아키텍처 패턴

## 폴더 구조

```
project/
  app/
    server.py      FastAPI: 업로드, 작업 생성·실행·조회, 설정, 정적 파일
    pipeline.py    Job 클래스 (액션 실행 엔진)
    <domain>.py    순수 함수 (분석·점수·변환)
    media.py       외부 바이너리 래퍼 (진행률 파싱)
    ai.py          AI 호출 + 도구별 프롬프트/스키마
    settings.py    설정 파일 + .env + 환경변수
    cli.py         process / watch / presets
    static/        index.html, style.css, app.js
  tests/           conftest.py가 작업 폴더 격리
  workflow.json    프리셋
  run.py           uvicorn 실행 + 브라우저 열기
  workspace/       (git 제외) uploads/, jobs/<id>/, settings.json
```

## Job: 액션 파이프라인

```python
LABELS = {  # 순서 = 실행 순서. "ai_" 접두사 = 키가 필요한 AI 액션
    "analyze": "분석", "transcribe": "음성 인식", "ai_fix_subs": "AI 자막 교정",
    "metadata": "메타데이터", "silence": "무음 제거", "shorts": "쇼츠", ...
}
AI_ACTIONS = {k for k in LABELS if k.startswith("ai_")}

class Job:
    def prepare(self, actions, options=None):
        # 옵션 병합 → 선행 단계 자동 추가 → stages 초기화 → job.json 저장
    def start(self, actions, options=None):     # 웹: 백그라운드 스레드
        self.prepare(actions, options); threading.Thread(target=self._run, daemon=True).start()
    def run_sync(self, actions, options=None):  # CLI·테스트: 동기
        self.prepare(actions, options); self._run(); return self.status == "done"
    def _run(self):
        for key in self.stages:            # 각 단계: running → _do_<key>() → done/skipped
            skipped = getattr(self, f"_do_{key}")()
        # 예외 → status=error, 로그에 트레이스백 (AIError는 메시지만)
        # finally: 요약·파일 목록 갱신, 저장
```

- `_do_<key>()`는 `self.result`에 결과를 넣고, 파일은 `self.dir`에 쓴다. 할 일이 없으면 `True`(건너뜀)를 반환한다.
- 결과는 **누적**된다. 같은 작업에 다른 액션을 나중에 추가로 실행할 수 있다.
- 다시 실행할 때는 이전 산출물을 지우고 새로 만든다 (예: `shorts_*`).
- 진행률은 `self.stages[key]["progress"]`에 0~1로 기록하고, 프론트가 폴링한다.
- 무거운 외부 프로세스(인코딩)는 전역 락으로 한 번에 하나만 돌린다.
- 대용량 중간 데이터(디코딩한 오디오 등)는 실행이 끝나면 메모리에서 해제한다.
- 작업 폴더 경로는 `CUTFLOW_WORKSPACE` 같은 **환경변수로 바꿀 수 있게** 한다 (테스트 격리용).

## 서버 API

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/api/status` | 기능 라벨, AI 액션 목록, 기본 옵션, 키 존재 여부(키 값은 절대 안 보냄) |
| POST | `/api/upload` | 확장자 허용 목록, 스트리밍 저장, 메타 조회 실패 시 삭제 |
| POST | `/api/jobs` | `{file_id, name, actions, options}` → 새 작업 시작 |
| POST | `/api/jobs/{id}/run` | 기존 작업에 액션 추가 실행 (실행 중이면 409) |
| GET | `/api/jobs`, `/api/jobs/{id}` | 목록은 디스크에서 새 작업도 다시 읽음 (CLI로 만든 작업 표시) |
| DELETE | `/api/jobs/{id}` | 실행 중이면 거부 |
| POST | `/api/settings` | 키는 **저장 전에 검증** |
| POST | `/api/settings/test` | 저장된 키 확인 |

## 보안 규칙 (반드시 지킬 것)

- 정적 공개는 **결과 폴더(`workspace/jobs`)만**. `workspace` 전체를 공개하면 `settings.json`의 API 키가 노출된다.
- `file_id`는 정규식으로 검증한다 (경로 조작 방지).
- 액션 이름은 허용 목록으로 검증한다.
- 키 없는 AI 액션은 서버에서 403으로 막는다 (프론트 잠금만으로는 부족하다).
- 공개 설정 응답에는 `has_api_key`와 앞뒤 몇 글자 힌트만 넣는다.
- 서버는 `127.0.0.1`에만 바인딩한다.

## 외부 바이너리 래퍼 (ffmpeg 예시)

- `imageio_ffmpeg.get_ffmpeg_exe()`로 번들된 ffmpeg를 쓴다. ffprobe는 없으므로 `ffmpeg -i`의 stderr를 파싱한다.
- 진행률: `-progress pipe:1 -nostats`의 `out_time_us`를 전체 길이로 나눈다. stderr는 별도 스레드로 읽어 버린다 (파이프가 막히는 것 방지).
- 컷 편집: `[0:v]split=N` → 각각 `trim` + `setpts=PTS-STARTPTS` → `concat=n=N:v=1:a=1`. 오디오도 `asplit`/`atrim`/`asetpts`로 같게 한다. 필터가 길어지므로 **`-filter_complex_script` 파일**로 넘긴다.
- 자막 번인: ASS 파일을 작업 폴더에 쓰고 `cwd=작업 폴더` + 상대 경로 `ass=file.ass`로 지정한다 (Windows 경로의 `C:` 이스케이프 문제 회피). 한글 폰트는 `Malgun Gothic`.
- 세로 쇼츠의 흐린 배경: `split → [bg] scale=increase,crop,gblur ; [fg] scale=decrease → overlay 가운데`.
- Windows에서는 `creationflags=CREATE_NO_WINDOW`로 콘솔 창이 뜨지 않게 한다.

## CLI 폴더 감시

- 원본은 먼저 `inbox/done/`으로 옮긴 뒤 그 경로에서 처리한다 (웹앱에서 다시 실행할 때도 그 경로를 쓴다).
- 실패하면 `inbox/failed/`로 옮기고 `<이름>_오류.txt`(오류 + 로그 끝부분)를 남긴다.
- 결과는 `outbox/<이름>_<날짜시각>/`에 복사하고 사람이 읽는 `요약.txt`를 만든다.
- `--once`: 지금 있는 파일만 처리하고 종료한다 (스케줄러·테스트용. 준비 상태 확인은 생략).
- 출력에 이모지·한글이 있으면 `sys.stdout.reconfigure(encoding="utf-8")`를 호출한다.
