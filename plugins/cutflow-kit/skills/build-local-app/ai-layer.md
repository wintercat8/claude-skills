# AI 기능 (API 키로 잠금 해제)

> Claude API 코드를 쓰기 전에 `claude-api` 스킬이 있으면 **반드시 먼저 읽는다.** 모델 ID, `thinking`/`output_config` 형식, fallback 파라미터는 자주 바뀐다. 아래는 CutFlow를 만들 때(2026-09) 쓴 형태이며, 스킬 문서와 다르면 스킬 문서를 따른다.

## 공통 호출 함수

```python
def call(api_key, prompt, schema, effort="medium", max_tokens=32000) -> dict:
    with anthropic.Anthropic(api_key=api_key).beta.messages.stream(
        model=MODEL,                                   # claude-api 스킬의 기본 모델
        max_tokens=max_tokens,
        betas=["server-side-fallback-2026-07-01"],
        fallbacks="default",                           # 정책 거절 시 서버가 다른 모델로 재시도
        thinking={"type": "adaptive"},
        output_config={"effort": effort, "format": {"type": "json_schema", "schema": schema}},
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        response = stream.get_final_message()
    # stop_reason 확인: "refusal" → 오류, "max_tokens" → 잘림 오류
    # 첫 text 블록을 json.loads
```

- **스트리밍**: 긴 출력(자막 전체 교정·번역)에서도 타임아웃이 나지 않는다.
- **JSON 스키마 출력**: 모든 object에 `additionalProperties: false`와 `required`를 넣는다.
- **예외 처리**: 넓은 예외 하나로 받지 말고 `AuthenticationError`, `PermissionDeniedError`, `RateLimitError`, `APIStatusError`, `APIConnectionError` 순서로 잡아, 사용자가 읽을 수 있는 한글 메시지로 바꾼다.
- **fallback 옵션**: 켰다는 사실을 사용자에게 알린다.

## 키 관리

- 우선순위: 설정 화면에서 저장한 키 → `.env` → 환경변수.
- **저장 전 검증**: `client.models.retrieve(MODEL)`은 무료이며, 잘못된 키면 401이 난다.
- 키 값은 로그, 응답, 파일 목록 어디에도 남기지 않는다. `.gitignore`에 `.env`와 설정 파일을 넣는다.

## 도구 설계 요령

- 입력은 앞 단계의 산출물(자막 등)을 쓴다. 자막이 없으면 음성 인식 단계를 자동으로 추가하고, 그래도 없으면 명확한 오류를 낸다.
- **줄 단위 작업**(교정·번역)은 `번호<TAB>텍스트`로 보내고 `{id, text}` 배열로 받는다. 받은 결과는 번호로 원본에 다시 맞춘다. 교정은 **바뀐 줄만** 받는다.
- **구간을 고르는 작업**(쇼츠 추천)은 받은 시작·끝 값을 검증한다 (범위 안, 최소 길이, 겹침 없음).
- **키가 없을 때 대안**: 규칙 기반 버전(키워드 추출, 템플릿)을 두면 키 없이도 앱이 쓸모 있다.

## 검증 (과금 없이)

1. 가짜 키로 호출해 **401 인증 오류**가 나면 요청 형식은 정상이다.
2. 테스트에서 `ai.call`을 monkeypatch해 스키마별 가짜 응답을 돌려주고, 모든 AI 액션의 파일 저장·재계산을 확인한다.
   - 가짜 응답은 **스키마 속성**으로 고른다 (프롬프트 문구로 고르면, 앞 단계 결과가 프롬프트에 섞여 오판한다).
3. 실제 키로 호출하는 테스트는 만들지 않는다. 실제 결과 품질은 사용자가 키를 넣고 확인하도록 안내한다.
