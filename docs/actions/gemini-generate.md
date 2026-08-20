## 1) `docs/actions/gemini-generate.md`


# Gemini Generate (Composite Action)

Gemini `generateContent` API를 호출해서 **생성된 text를 GitHub Actions output(`text` / `text_file`)으로 반환**한다.

- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/<MODEL>:generateContent`
- Auth: `x-goog-api-key` 헤더

---

## ⚠ 128KiB 제약 (반드시 읽을 것)

리눅스 `MAX_ARG_STRLEN` = **131,072 bytes(128KiB)** 는 **단일 env 값 / 단일 argv 문자열**의 상한이다.
이걸 넘기면 `execve`가 `E2BIG`을 반환하고, 러너는 스텝 쉘 자체를 띄우지 못한다:

```
An error occurred trying to start process '/usr/bin/bash' with working directory '...'.
Argument list too long
```

action input 은 결국 `env` 로 전달되므로 `content` / `system_instruction` 에 128KiB 를 넘는 값을 주면
**액션 내부 스텝이 시작조차 못 한다.** 대용량 페이로드는 **`content_file` / `system_instruction_file`** 을 쓴다.

| 상황 | 쓸 입력 |
|---|---|
| 프롬프트가 작다 (수 KB) | `content` / `system_instruction` |
| 커밋 로그·diff·문서 전량처럼 크다 | **`content_file` / `system_instruction_file`** |
| 응답이 클 수 있다 (`max_output_tokens` 큼) | 결과는 **`text_file`** 로 받는다 |

`text` output 은 `GITHUB_OUTPUT`(1MB 한계)을 통과하지만, **그 값을 다시 다른 스텝의 `env` 로 넘기면
같은 128KiB 한계에 걸린다.** 대용량 응답에서는 `text_file` 경로를 넘기고 받는 쪽에서 파일을 읽어야 한다.

---

## Contract

### Requires
- runner에 `jq` 필요 (GitHub-hosted ubuntu-latest에는 기본 존재, `--rawfile` 사용)
- `curl` 필요 (기본 존재)
- 작업 파일은 `$RUNNER_TEMP` 하위에 만든다 (미설정 시 `/tmp`)

### Outputs
- `text` — 생성 텍스트 본문. `steps.<id>.outputs.text` (배치 모드에서는 비어 있다)
- `text_file` — 생성 텍스트가 담긴 파일 경로. `steps.<id>.outputs.text_file` (**대용량 권장**)
- `text_dir` — 배치 모드 출력 디렉토리. 입력 파일과 같은 이름으로 쓰인다
- `call_count` — 실제 API 호출 횟수

---

## ⚠ 출력 상한 — 사고 토큰이 예산을 함께 쓴다

2.5 계열은 thinking 모델이고 **사고 토큰이 `max_output_tokens` 예산을 함께 소비한다.**
입력이 크고 지시가 복잡하면 사고가 예산을 다 먹고 본문이 잘린다. 실측:

| thinking_budget | max_output_tokens | 사고 | 본문 | finishReason |
|---|---|---|---|---|
| 미지정 | 50,000 | 46,554 | 3,442 | `MAX_TOKENS` |
| 16,384 | 50,000 | 16,383 | 33,613 | `MAX_TOKENS` |
| 12,288 | 65,535 | 12,285 | 53,246 | `MAX_TOKENS` |

`finishReason != STOP` 이면 액션이 **실패시킨다.** 잘린 결과가 성공으로 하류에
흘러가지 않는다. 매 호출 `usageMetadata` 를 로그로 남기므로 원인(사고 vs 본문)이
바로 갈린다.

출력 상한을 넘길 분량이면 `content_dir` **배치 모드**로 입력을 나눈다.

---

## Inputs

### `api_key` (required)
Gemini API Key (GitHub Secrets로 전달)

### `model` (required)
모델 ID
- 예: `gemini-2.5-flash`

### `content` (optional, default: `""`)
유저 프롬프트 본문(최종 입력 텍스트), inline 전달.
- **128KiB 미만만 가능.** 초과 시 `content_file` 사용
- `content_file` 이 지정되면 무시된다

### `content_file` (optional, default: `""`)
유저 프롬프트 본문이 담긴 **파일 경로**.
- 지정하면 `content` 를 무시한다
- 파일이 없으면 `::error::content_file not found: <path>` 로 실패
- `content` / `content_file` 둘 다 비면 `::error::content is empty.` 로 실패

### `content_dir` (optional, default: `""`)
**배치 모드.** 이 디렉토리의 파일을 정렬 순서대로 하나씩 **별도 호출**한다.

- 지정하면 `content` / `content_file` 을 무시한다
- 출력은 `text_dir` 에 **같은 파일명**으로 쓰인다. `text` / `text_file` 은 비어 있다
- 호출마다 재시도·HTTP/에러 분기·`finishReason` 잘림 가드·`usageMetadata` 로그가
  단건 모드와 동일하게 적용된다. 로그는 `batch <파일명>:` 접두로 구분된다
- 한 호출로 출력 상한을 넘기는 분량을 나눠 처리할 때 쓴다.
  배치 순서가 곧 결과 순서이므로 파일명을 `batch_001`, `batch_002` … 처럼 정렬 가능하게 둔다

```yaml
- uses: KPNP-R-D/github-actions-module/actions/gemini-generate@gemini-generate/v1
  id: compact
  with:
    api_key: ${{ secrets.GEMINI_API_KEY }}
    model: gemini-2.5-flash
    system_instruction_file: ${{ steps.batches.outputs.compact_system_file }}
    content_dir: ${{ steps.batches.outputs.batch_dir }}
    max_output_tokens: "65535"
    thinking_budget: "8192"
# steps.compact.outputs.text_dir / call_count
```

### `temperature` (optional, default: `"0.1"`)
generationConfig.temperature
- action 내부에서 `--argjson`으로 주입되므로 **숫자 문자열**이어야 한다.
  - 예: `"0.3"`

### `max_output_tokens` (optional, default: `"2048"`)
generationConfig.maxOutputTokens
- action 내부에서 `--argjson`으로 주입되므로 **정수 문자열**이어야 한다.
  - 예: `"50000"`
- 크게 잡을수록 응답이 128KiB 를 넘길 확률이 오르므로 `text_file` 사용을 권장한다
- 크게 잡으면 응답 시간도 길어진다 → `max_time` 을 함께 올려야 한다

### `max_time` (optional, default: `"600"`)
`curl --max-time` (초). 요청 1회의 전체 상한.

- **대용량 프롬프트에서 가장 흔한 실패 원인이다.**
  프롬프트 148KB(약 40K 토큰) + `max_output_tokens: 50000` 조합은 60초로는 끝나지 않는다
- 타임아웃하면 `::error::HTTP request failed (network/timeout).` 로 실패한다.
  로그의 `max_time` / `retry` / `body` bytes 를 보고 올린다

### `retry` (optional, default: `"3"`)
`curl --retry` 횟수.

- `--retry-all-errors` 이므로 **타임아웃과 5xx 모두 재시도 대상**이다.
  실제로 Gemini `503 UNAVAILABLE`("This model is currently experiencing high demand") 이 관측된다
- `--retry-delay` 를 주지 않아 curl 기본 **지수 백오프**(1s, 2s, 4s ...)로 동작한다.
  503 과부하는 고정 1초 간격 재시도로는 잘 안 풀린다
- `--retry-max-time ${max_time}` 으로 재시도 시작 창이 묶여, `retry` 값과 무관하게
  최악 대기시간은 `2 * max_time` 이다
- 단, **느려서 타임아웃하는 경우 재시도는 도움이 안 된다** — 그 경우엔 `max_time` 을 올려야 한다

### `system_instruction` (optional, default: `""`)
system instruction 텍스트, inline 전달.
- 비어있지 않으면 request body에 `systemInstruction.parts[0].text`로 포함된다
- `content` 와 동일하게 128KiB 제약을 받는다
- `system_instruction_file` 이 지정되면 무시된다

### `system_instruction_file` (optional, default: `""`)
system instruction 이 담긴 **파일 경로**.
- 지정하면 `system_instruction` 을 무시한다
- 파일이 없으면 실패, 파일이 비어 있으면 systemInstruction 없이 요청한다

---

## Request payload behavior

body 는 `jq --rawfile` 로 **파일에서 직접** 읽어 `$RUNNER_TEMP/gemini_body.json` 에 쓰고,
`curl --data-binary @<file>` 로 전송한다. (`-d "$BODY"` 는 argv 128KiB 한계에 걸린다)

### system_instruction이 있는 경우
```json
{
  "systemInstruction": { "parts": [ { "text": "<system_instruction>" } ] },
  "contents": [ { "role": "user", "parts": [ { "text": "<content>" } ] } ],
  "generationConfig": {
    "temperature": <temperature>,
    "maxOutputTokens": <max_output_tokens>
  }
}
````

### system_instruction이 비어있는 경우

```json
{
  "contents": [ { "role": "user", "parts": [ { "text": "<content>" } ] } ],
  "generationConfig": {
    "temperature": <temperature>,
    "maxOutputTokens": <max_output_tokens>
  }
}
```

---

## Error handling

### 1) 입력 오류

* `content_file` / `system_instruction_file` 경로가 없으면 `::error::... not found: <path>`
* content 가 비면 `::error::content is empty. Pass either 'content' or 'content_file'.`

### 2) 네트워크 / HTTP

`curl` 은 `--connect-timeout 10 --max-time ${max_time} --retry ${retry} --retry-max-time ${max_time} --retry-all-errors` 로 호출한다.
`--retry-delay` 를 주지 않아 curl 기본 **지수 백오프**(1s, 2s, 4s ...)가 동작하고,
`--retry-max-time` 이 재시도 시작 창을 묶어 최악 대기시간은 `2 * max_time` 이다.

* curl 자체 실패(`http_code` 가 비거나 `000`) → `::error::HTTP request failed (network/timeout).`
* 2xx 아님 → `::error::HTTP <code> from Gemini API. Body: <응답 앞 2000 bytes>`

### 3) API error

응답에 `.error`가 존재하면 상태별로 분기해 실패 처리:

* `429` / `RESOURCE_EXHAUSTED` → quota/rate limit
* `400` / `INVALID_ARGUMENT` → invalid request
* `401` / `UNAUTHENTICATED` → auth failed
* `403` / `PERMISSION_DENIED` → permission denied
* 그 외 → `::error::Gemini API error (<code>/<status>): <message>`

### 4) Empty text

`candidates` 가 없거나 parts 텍스트가 전부 비면 실패:

* 로그: `::error::Gemini returned empty text. finishReason=<reason>. Raw: <응답 앞 2000 bytes>`
* exit 1

> 에러 로그에 응답을 전량 덤프하지 않고 앞 2000 bytes 만 남긴다 (annotation 한계 + argv 한계 회피).

---

## Outputs (GITHUB_OUTPUT)

응답 텍스트는 먼저 `$RUNNER_TEMP/gemini_text.txt` 로 쓰고, 경로와 본문을 함께 내보낸다.
멀티라인 안전을 위해 `text` 는 랜덤 delimiter 로 기록한다.

```bash
echo "text_file=${text_path}" >> "$GITHUB_OUTPUT"

DELIM="__EOF_$(date +%s%N)__"
{
  echo "text<<$DELIM"
  cat "${text_path}"
  echo "$DELIM"
} >> "$GITHUB_OUTPUT"
```

파트가 여러 개면 `\n` 으로 join 한다.

---

## Usage

### Basic (작은 프롬프트)

```yaml
- name: Gemini generate
  id: llm
  uses: KPNP-R-D/github-actions-module/actions/gemini-generate@gemini-generate/v1
  with:
    api_key: ${{ secrets.GEMINI_API_KEY }}
    model: gemini-2.5-flash
    content: ${{ steps.prompt.outputs.content }}
```

### 대용량 (파일 경유 — 권장)

```yaml
- name: Gemini generate
  id: llm
  uses: KPNP-R-D/github-actions-module/actions/gemini-generate@gemini-generate/v1
  with:
    api_key: ${{ secrets.GEMINI_API_KEY }}
    model: gemini-2.5-flash
    system_instruction_file: ${{ steps.prompt.outputs.system_file }}
    content_file: ${{ steps.prompt.outputs.content_file }}
    temperature: "0.3"
    max_output_tokens: "50000"
    max_time: "600"   # 150KB 프롬프트 + 50K 출력 토큰은 60초로 부족
    retry: "3"
```

결과 사용 — **본문을 env 로 넘기지 말고 경로를 넘긴다**:

```yaml
- name: Use output
  env:
    LLM_TEXT_FILE: ${{ steps.llm.outputs.text_file }}
  run: wc -c < "$LLM_TEXT_FILE"
```

작은 응답이면 기존처럼 써도 된다:

```yaml
- name: Use output
  run: echo "${{ steps.llm.outputs.text }}"
```

---

## Notes / Pitfalls

* `temperature`, `max_output_tokens`는 jq `--argjson`으로 넣기 때문에 **숫자여야 함**.

  * `"abc"` 같은 값이면 jq 단계에서 실패한다.
* **`content` / `system_instruction` / `text` 를 env 로 릴레이할 때 128KiB 한계를 항상 의식한다.**
  대용량이면 `*_file` / `text_file` 을 쓴다.
* 응답 파싱은 첫 후보의 parts 전체를 `\n` 으로 join 한다:

  * `.candidates[0].content.parts[] | .text`
  * 모델/응답 포맷이 바뀌면 보강 필요.
* 작업 파일은 `$RUNNER_TEMP` 하위이므로 **같은 job 내 스텝 간에만** 공유된다.
  job 이 다르면 artifact 등 다른 수단이 필요하다.
