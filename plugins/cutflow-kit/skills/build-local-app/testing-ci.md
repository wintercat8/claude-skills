# 테스트와 CI

## conftest.py: 격리가 먼저

```python
import os, shutil, tempfile
from pathlib import Path

_TMP = Path(tempfile.mkdtemp(prefix="app-test-"))
os.environ["APP_WORKSPACE"] = str(_TMP / "workspace")   # 앱이 이 변수로 작업 폴더를 정하게 해 둔다
os.environ["ANTHROPIC_API_KEY"] = ""                    # .env에 키가 있어도 '키 없음'으로 시작

import pytest                                           # app은 반드시 이 아래에서 import

@pytest.fixture
def api_key(monkeypatch):                               # AI 테스트용 가짜 키
    fake = lambda: {"anthropic_api_key": "sk-ant-test"}
    monkeypatch.setattr(settings, "load_settings", fake)
    monkeypatch.setattr(pipeline, "load_settings", fake)  # 이름으로 import한 모듈마다 패치

def pytest_sessionfinish(session, exitstatus):
    shutil.rmtree(_TMP, ignore_errors=True)
```

`pytest.ini`:

```ini
[pytest]
testpaths = tests
pythonpath = .
```

## 테스트 입력은 코드로 생성

바이너리 샘플 파일을 저장소에 넣지 않는다. ffmpeg lavfi로 매번 만든다.

```python
audio = "aevalsrc='sin(2*PI*440*t)*between(mod(t\\,4)\\,0\\,2.5)*(0.25+0.6*between(t\\,8\\,14))':s=44100:d=24"
# 4초 주기: 2.5초 소리 + 1.5초 무음, 8~14초 구간은 크게 → 무음 감지·하이라이트 둘 다 검증 가능
ffmpeg -f lavfi -i testsrc2=size=640x360:rate=25:d=24 -f lavfi -i <audio> -c:v libx264 -preset ultrafast ...
```

- 순수 함수 테스트는 numpy 배열로 직접 만든다 (톤 + 무음 이어 붙이기).
- 사실적인 음성 샘플(수동 검증용)은 Windows `System.Speech`로 만든다. `Microsoft Heami`가 한국어 음성이다.

## 테스트 구성 (약 30개, 10초 안팎)

| 파일 | 내용 |
|---|---|
| `test_<domain>.py` | 순수 함수: 경계값, 빈 입력, 시간 재매핑, 포맷 |
| `test_pipeline.py` | 실제 바이너리 실행. 출력 길이·해상도, 액션 재실행 시 이전 산출물 교체, 키 없이 AI 액션 실패, 가짜 응답으로 AI 액션 |
| `test_server.py` | TestClient: 업로드 거부, 액션 하나 실행 후 추가 실행, 403/400, **설정 파일이 공개되지 않는지** |
| `test_cli.py` | 저장소의 workflow.json 유효성, 키 없을 때 AI 건너뜀, `process`, `watch --once`(done/ 이동, 무관한 파일 무시), 깨진 파일 → failed/ |

## GitHub Actions

```yaml
name: CI
on: { push: {}, pull_request: {}, workflow_dispatch: {} }
concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }
jobs:
  test:
    runs-on: ${{ matrix.os }}
    timeout-minutes: 20
    strategy:
      fail-fast: false
      matrix:
        include:
          - { os: ubuntu-latest,  python: "3.12" }
          - { os: windows-latest, python: "3.14" }   # 실제 사용 환경
    defaults: { run: { shell: bash } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "${{ matrix.python }}", cache: pip, cache-dependency-path: requirements.txt }
      - run: |
          python -m pip install --upgrade pip
          grep -v '^faster-whisper' requirements.txt > ci-requirements.txt   # 무거운 선택 의존성 제외
          pip install -r ci-requirements.txt pytest httpx
      - run: |
          python -m compileall -q app tests
          node --check app/static/app.js
      - run: python -m pytest -v
```

## 저장소 위생

- `.gitignore`: `.env`, `.env.*`, `!.env.example`, 작업 폴더, `inbox/`, `outbox/`, `__pycache__/`, 미디어 확장자, `.claude/settings.local.json`
- `.gitattributes`: `*.bat`와 `*.ps1`은 `eol=crlf`. `.ps1`에 한글이 있으면 **UTF-8 BOM**으로 저장한다 (Windows PowerShell 5.1이 BOM 없는 파일을 ANSI로 읽는다).
- push 전 비밀 값 스캔: `git grep --cached -nIE "sk-ant-[A-Za-z0-9_-]{20,}"` (종료 코드 1 = 없음).
- 비공개 저장소의 CI 결과는 로그인 없이 볼 수 없다. 사용자에게 Actions 탭 확인을 요청한다.
