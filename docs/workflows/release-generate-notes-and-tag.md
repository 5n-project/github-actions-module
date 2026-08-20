# Release · Generate Notes & Tag (Reusable)

선택된 서비스들의 커밋 로그를 수집해 Draft를 만들고, Gemini로 릴리즈 노트를 정제한 뒤, Notion 페이지 생성 + `bundle/<releaseKey>` 태그 생성/푸시 + GitHub Release 생성까지 처리한다.

---

## Contract

### Required permissions

Caller에서:

* `permissions: contents: write`

  * 태그 생성/푸시, release 생성에 사용

### Required checkout

* `actions/checkout@v4` with `fetch-depth: 0`

  * 태그/히스토리 기반 previousTag 계산 + 범위 log 추출 때문에 필요

---

## workflow_call inputs

### Required

* `catalog_path` (string)
  서비스 카탈로그 JSON 경로
  e.g. `.github/release/service-catalog.json`
* `prompts_dir` (string)
  프롬프트 루트 디렉토리
  e.g. `prompts`
* `inputs_json` (string)
  caller의 `toJson(inputs)` 결과(서비스 체크박스 읽는 용도)
* `notion_category` (string)
  Notion DB 속성 `구분`의 select 값 (예: `백엔드`)

### Optional

* `manifest_dir` (string, default: `releases`)

  * 현재 워크플로에서는 **YML 파일을 생성하지 않음**. (입력 정의는 유지)
* `compact_batch_size` (string, default: `"150"`)
  PASS 1(커밋 메시지 정제) 배치 1건에 담을 커밋 수.
  실측 커밋당 출력 약 104 토큰. 150이면 배치당 약 16K 토큰으로 본문 예산(약 57K)에 3배 이상 여유가 있다.
  값을 줄이면 호출 수가 늘어 무료 티어 분당 한도(429)에 닿기 쉽다.
  아래 “출력 상한과 2-pass” 참고.
* `current_tags` (string, default: `""`)
  선택된 서비스들의 현재 배포 버전 매핑. 사실상 필수값.
  형식: `key=value,key=value` (CSV)
* `release_key` (string, default: `""`)
* `note` (string, default: `""`)

---

## workflow_call secrets

Required:

* `GEMINI_API_KEY`
* `NOTION_TOKEN`
* `NOTION_DB_ID`

Optional:

* `NOTION_DS_NAME`

---

## Catalog JSON spec

`catalog_path`는 아래 형태(JSON object)를 기대한다.

```json
{
  "Admin": {
    "inputKey": "admin",
    "tagPrefix": "kpnp-admin",
    "servicePaths": ["service/kpnp-admin"],
    "sharedPaths": ["service/core"]
  },
  "Champs": {
    "inputKey": "champs",
    "tagPrefix": "kpnp-champs",
    "servicePaths": ["service/kpnp-champs"],
    "sharedPaths": ["service/core", "service/security"]
  }
}
```

* `inputKey`: caller workflow_dispatch 체크박스 key
* `tagPrefix`: 태그 prefix. 태그 목록은 `"<tagPrefix>/v*"` 패턴으로 조회
* `servicePaths`: 서비스별 커밋 로그 대상 경로
* `sharedPaths`: 공통 모듈 커밋 로그 대상 경로(선택 서비스들의 sharedPaths merge)

---

## current_tags mapping (핵심 입력)

형식: CSV `key=value`

* key는 `서비스명` 또는 `inputKey` 둘 다 허용.
* value는 **태그** 또는 **git revision(커밋 SHA/브랜치/HEAD 등)** 허용.
* 검증:

  * value가 tag로 존재하면 tag 모드
  * 아니면 commit으로 해석 가능한 revision이면 commit 모드
  * 둘 다 아니면 실패
* 선택하지 않은 서비스가 포함되면 실패
* 선택된 서비스가 누락되면 실패

예시:

* 태그로 지정

  * `Admin=kpnp-admin/v1.2.3,Champs=kpnp-champs/v2.0.1`
  * `admin=kpnp-admin/v1.2.3,champs=kpnp-champs/v2.0.1`
* 커밋/리비전으로 지정

  * `admin=1a2b3c4d...`
  * `champs=origin/main`

---

## What it does

### Outputs / Side effects

* Step Summary에:

  * included services / shared paths / baseline HEAD / globalPrevSha / 서비스별 previousTag
  * Shared+Services git log 커맨드/결과 디버그
  * Gemini 결과
* Notion 페이지 생성 + 본문 블록 append + Raw Draft(toggle) append

  * `page_id`를 step output으로 제공 (`steps.notion_draft.outputs.page_id`)
* Git tag 생성/푸시:

  * `bundle/<releaseKey>` annotated tag (idempotent + commit mismatch 검증)
* GitHub Release 생성:

  * tag=`bundle/<releaseKey>`로 release 생성 (`gh release create --generate-notes`)
  * 이미 존재하면 스킵

---

## 대용량 페이로드 취급 (128KiB 한계)

리눅스 `MAX_ARG_STRLEN` = **131,072 bytes(128KiB)** 는 **단일 env 값 / 단일 argv 문자열**의 상한이다.
초과하면 `execve`가 `E2BIG`을 반환하고 러너가 스텝 쉘 자체를 못 띄운다:

```
An error occurred trying to start process '/usr/bin/bash' with working directory '...'.
Argument list too long
```

이 워크플로는 커밋 **본문 전량**(`%B`)을 draft 로 만들기 때문에 실측 **148KB** 를 넘긴 사례가 있다.
`git log -n 80` 상한은 커밋 **개수** 제한이라 바이트 상한이 아니다 —
80커밋 × 평균 1.8KB 만으로 이미 141KB 다.

그래서 대용량 값은 **본문을 output/env 로 릴레이하지 않고 `$RUNNER_TEMP` 파일에 쓰고 경로만 넘긴다.**

| 값 | 전달 방식 |
|---|---|
| draft (커밋 로그 전량) | `steps.build_draft.outputs.draft_file` → `$RUNNER_TEMP/release_draft.md` |
| Gemini system prompt | `steps.prompt.outputs.system_file` → `$RUNNER_TEMP/gemini_system.md` |
| Gemini content (prompt + 커밋 로그) | `steps.prompt.outputs.content_file` → `$RUNNER_TEMP/gemini_content.md` |
| Gemini 응답 | `steps.llm.outputs.text_file` → `$RUNNER_TEMP/gemini_text.txt` |
| Notion 본문 | `DRAFT_FILE` / `RELEASE_NOTES_FILE` env (경로) |

작게 유지되는 값(`manifest_json` 약 1.5KB, `has_commits`, `commit_count`,
`service_commit_counts`, `releaseKey`, `page_id`)은 그대로 output/env 로 넘긴다.

**이 워크플로를 수정할 때 지켜야 할 것:**

* 커밋 로그·프롬프트·LLM 응답 계열 값을 `env:` 나 action `with:` 에 **본문으로** 넣지 않는다
* `curl -d "$BODY"` 같은 argv 전달도 같은 한계에 걸린다 → `--data-binary @<file>` 을 쓴다
* `$RUNNER_TEMP` 는 **같은 job 안에서만** 공유된다 (이 워크플로는 단일 job `start` 이라 문제없음)

---

## 출력 상한과 2-pass

128KiB 문제(입력)를 해소하면 곧 **출력 상한**(모델이 한 번에 쓸 수 있는 양)에 부딪힌다.

`gemini-2.5-flash` 는 thinking 모델이고 **사고 토큰이 `maxOutputTokens` 예산을 함께 쓴다.**
커밋 322개(prompt 151,877 토큰)를 한 번에 넘긴 실측:

| thinkingBudget | maxOutputTokens | 사고 | 본문 | 본문 크기 | finishReason |
|---|---|---|---|---|---|
| 기본(미지정) | 50,000 | 46,554 | 3,442 | 11,949 B | `MAX_TOKENS` |
| 16,384 | 50,000 | 16,383 | 33,613 | 101,763 B | `MAX_TOKENS` |
| 12,288 | **65,535** (모델 상한) | 12,285 | 53,246 | 164,874 B | `MAX_TOKENS` |

65,535 는 모델 출력 상한이라 더 올릴 수 없고, **출력 분량은 커밋 수에 비례**한다.
그래서 호출을 나눈다.

```
PASS 1 (배치 N회)  커밋을 compact_batch_size 개씩 나눠 "커밋당 몇 줄"로 압축
                   -> 배치당 출력 5~8K 토큰. 상한에 여유가 크다
PASS 2 (1회)       압축본 전체 + TEMPLATE -> 릴리즈 노트 문서 1개
```

PASS 2 는 **caller 의 기존 프롬프트를 그대로** 쓴다. 입력만 원본 draft 대신 압축본으로
바뀌므로 문서 병합 파서가 필요 없고 Summary 도 한 문서 안에서 생성된다.
`release-notes.compact.md` 를 두지 않은 caller 는 워크플로 내장 기본 프롬프트로 동작한다.

실측(커밋 322개 · draft 475,771 bytes · `compact_batch_size: 80`(당시 기본값)):

* PASS 1: 5회 호출, 전부 `finishReason=STOP`, 출력 합계 95,783 bytes (draft 대비 80% 압축)
* PASS 2: prompt 31,446 · 본문 13,792 · 사고 12,284 = 57,522 / 65,535 → `STOP`
* 최종 릴리즈 노트 45,491 bytes

**잘림은 조용히 지나가지 않는다.** 액션이 `finishReason != STOP` 이면 실패시키므로,
부분 응답이 Notion 까지 올라가는 경로는 막혀 있다.

---

<details>
<summary><strong>Steps detail</strong></summary>

### 0) Checkout

* `fetch-depth: 0`로 태그/히스토리 전체 확보

### 1) Install python deps

* `pyyaml` 설치 (현재 워크플로는 manifest를 JSON으로 다루지만 공용 의존성 유지)

### 2) Resolve releaseKey

* `inputs.release_key`가 있으면 그대로 사용
* 없으면 `YYYY-MM-DD-<run_number 3자리>`로 생성

### 3) Create manifest (compute config only; no yml)

YML 파일을 만들지 않고 **manifest JSON**을 계산해 step output으로 export.

* `inputs_json`에서 선택 서비스(inputKey=true) 목록 생성
* `current_tags` 파싱 + 서비스별 current(tag/commit) 확정
* 서비스별 `previousTag` 계산

  * current가 tag면: `tagPrefix/v*` 리스트에서 현재 tag의 **바로 이전 tag**
  * current가 commit이면: `previousTag` 없음(= fallback 대상)
* 선택 서비스들의 `sharedPaths` merge → `shared.paths`
* `globalPrevSha` 선정(Shared 범위용)

  * 서비스별 후보를 모아 **timestamp가 가장 오래된 sha** 선택
  * 후보 규칙:

    1. `previousTag` 있으면 그 tag의 sha
    2. 없으면 `servicePaths`에서 `git log -n 80`의 “가장 오래된 커밋 sha”
* Step Summary에 included/shared/baseline/global prev/previous tags 출력
* 결과: `steps.manifest.outputs.manifest_json`

### 4) Build draft (commit logs)

* `git fetch --tags --force`
* manifest JSON에서 `baselineHeadSha/globalPrevSha/shared.paths/services` 읽음
* Shared logs 범위:

  * globalPrevSha 있으면 `globalPrevSha..baseline`
  * 없으면 `baseline`
* Service logs 범위:

  * currentType=tag:

    * previousTag 있으면 `previousTag..currentTag`
    * 없으면 `git log -n 80 currentTag`
  * currentType=commit:

    * `git log -n 80 currentRef` (정규화된 full sha)
* 커밋 메시지 파싱:

  * subject + body를 bullet로 변환
  * body는 indent 기반으로 1~3 depth 유지
  * `[RN-BOT]` 포함 커밋 제외
* output:

  * `draft_file`(= `$RUNNER_TEMP/release_draft.md` 경로), `has_commits`, `commit_count`, `service_commit_counts`
  * **draft 본문은 output/env 로 내보내지 않는다** — 아래 “대용량 페이로드 취급” 참고
* Step Summary에 debug(cmd/output + draft)를 `<details>`로 append

### 5) Fail job if no commit messages

* `has_commits != true`면 `exit 1` (빈 릴리즈 차단)

### 6) Build compact batches (PASS 1 입력)

Gemini 호출은 **2-pass** 다. 이유는 아래 “출력 상한과 2-pass” 참고.

* `draft_file` 을 `(섹션, 커밋 블록)` 단위로 쪼갠다
  (커밋 블록 = 커밋 제목 줄 + 뒤따르는 들여쓴 본문 줄 전체)
* `compact_batch_size` 개씩 배치로 나눠 `$RUNNER_TEMP/gemini_batches/batch_NNN.md` 로 쓴다.
  배치마다 해당 섹션 헤더(`## Shared` 등)를 다시 얹으므로 배치만 봐도 그룹을 알 수 있다
* PASS 1 프롬프트 결정:

  * `${prompts_dir}/gemini/release-notes.compact.md` 가 있으면 그것을 사용
  * 없으면 **워크플로 내장 기본값**을 사용 → 기존 caller 는 파일 추가 없이 동작한다
* output: `batch_dir`, `compact_system_file`, `batch_count`, `commit_count`

### 7) Gemini compact (PASS 1)

* `actions/gemini-generate` 를 **배치 모드**로 호출 (`content_dir: batch_dir`)
* 액션이 디렉토리의 파일을 정렬 순서대로 하나씩 개별 호출하고,
  각 호출마다 재시도·에러 분기·`finishReason` 잘림 가드를 동일하게 적용한다
* PASS 1 은 커밋 **제목을 원문 그대로** 남긴다 —
  PASS 2 의 그룹 매핑이 제목 prefix(`[Scope]` / `type(scope):`)에 의존한다
* 결과: `steps.compact.outputs.text_dir` (배치별 출력), `call_count`

### 8) Build Gemini content (PASS 2)

* `${prompts_dir}`에서 다음 파일 로드:

  * `gemini/release-notes.system.md`
  * `gemini/release-notes.input.md`
  * `release-notes.template.md`
* `{{COMMITS}}` 는 **PASS 1 배치 출력물을 순서대로 이어붙인 압축본**으로 치환,
  `{{TEMPLATE}}` 는 template 파일로 치환
* 즉 caller 프롬프트 계약은 그대로다. 입력만 원본 draft 대신 압축본으로 바뀐다
* 결과를 파일로 기록하고 경로만 output:

  * `steps.prompt.outputs.system_file` (= `$RUNNER_TEMP/gemini_system.md`)
  * `steps.prompt.outputs.content_file` (= `$RUNNER_TEMP/gemini_content.md`)

### 9) Gemini refine (PASS 2)

* `actions/gemini-generate` 단건 호출 (`if: steps.prompt.outputs.content_file != ''`)

  * `system_instruction_file = system_file`
  * `content_file = content_file`
  * `max_output_tokens: 65535` / `thinking_budget: "12288"` — 사고 토큰도
    `maxOutputTokens` 예산을 함께 쓴다. 아래 “출력 상한과 2-pass” 참고
  * `max_time: "600"` / `retry: "3"` — 응답까지 수 분이 걸린다. 액션 기본값 60초로는 타임아웃한다
* 결과: `steps.llm.outputs.text_file` (본문은 `steps.llm.outputs.text` 로도 나오지만 릴레이에는 쓰지 않는다)

### 10) Append Gemini output to Step Summary

* `text_file` 경로를 받아 파일을 읽어 Step Summary에 기록

### 11) Create Notion page (draft)

* DB에서 `data_source_id` resolve

  * `NOTION_DS_NAME` 있으면 name 매칭, 없으면 첫 data_source
* “배포 버전” rich_text

  * 서비스별 `current` 표시
  * commit이면 full sha를 7자리로 축약해 code 스타일로 표시
* “프로젝트” multi_select

  * includedServices 그대로
* Notion page 생성 properties:

  * 제목=releaseKey, 구분=notion_category, 배포 날짜=today, 배포 버전, 프로젝트
* 본문 append (`DRAFT_FILE` / `RELEASE_NOTES_FILE` 경로를 받아 파일에서 읽는다)

  * 데이터2(=Gemini) 상단 append (hr `---` 제거)
  * divider 1개
  * toggle `Raw Draft (Data 1)` 생성
  * toggle children에 데이터1(=draft) append
  * chunk 단위로 PATCH — Notion 상한(children <= 100, **중첩 포함** 전체 블록 <= 1000)을
    모두 만족하도록 재귀로 세어 나눈다
* output: `steps.notion_draft.outputs.page_id`

### 12) Create & push bundle tag (idempotent + verify)

* tag: `bundle/<releaseKey>`
* 존재하면:

  * tag가 가리키는 commit != HEAD → 실패(충돌 방지)
  * 같으면 notice 후 스킵
* 없으면:

  * annotated tag 생성 + push

### 13) Create GitHub Release (generate notes)

* `gh release view <tag>`로 존재 여부 확인
* 없으면 `gh release create <tag> --title ... --generate-notes`
* 있으면 notice 후 종료

</details>

---

## Caller example

```yaml
jobs:
  call-release-generate-notes-and-tag:
    uses: KPNP-R-D/github-actions-module/.github/workflows/release.reusable.yml@v1
    with:
      catalog_path: .github/release/service-catalog.json
      prompts_dir: prompts
      manifest_dir: releases

      inputs_json: ${{ toJson(inputs) }}
      current_tags: ${{ inputs.current_tags }}

      release_key: ${{ inputs.release_key }}
      note: ${{ inputs.note }}

      notion_category: "백엔드"
    secrets:
      GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
      NOTION_TOKEN: ${{ secrets.KPNP_NOTION_TOKEN }}
      NOTION_DB_ID: ${{ secrets.KPNP_NOTION_DB_ID }}
      NOTION_DS_NAME: ${{ secrets.KPNP_NOTION_DS_NAME }}
```

Caller workflow에는 아래도 같이 있어야 함:

```yaml
permissions:
  contents: write
```

---

## Troubleshooting

### `current_tags` 매핑 에러

* `current_tags`가 빈 값이면 실패.
* 선택 서비스 누락/선택되지 않은 서비스 포함/중복 key면 실패.
* key는 `서비스명` 또는 `inputKey`만 허용.

### tag/commit 조회 실패

* `fetch-depth: 0` 확인
* `current_tags`의 value가 실제 tag 또는 commit rev로 해석 가능해야 함

### Notion data_source_id resolve 실패

* `NOTION_DB_ID`가 올바른 DB인지 확인
* `NOTION_DS_NAME` 사용 시 정확히 일치해야 함

### 빈 릴리즈로 실패(`Fail job if no commit messages`)

* 범위 내 커밋이 없거나 `[RN-BOT]` 필터로 전부 제외된 케이스.
* Step Summary의 debug(각 서비스 git log cmd/출력)로 범위 확인.

### `Argument list too long` (E2BIG)

```
An error occurred trying to start process '/usr/bin/bash' with working directory '...'.
Argument list too long
```

* 스텝 **내용**이 아니라 스텝 **기동**이 실패한 것이다 — 단일 env/argv 값이 128KiB 를 넘었다.
* 실패한 스텝의 로그에서 `##[group]Run ...` 하단 env 에코를 보고 어떤 값이 큰지 찾는다.
* 해당 값을 `$RUNNER_TEMP` 파일로 옮기고 경로만 넘긴다. 위 “대용량 페이로드 취급(128KiB 한계)” 참고.
* 호출 측(`current_tags`, catalog `sharedPaths`)으로는 실질적으로 해소되지 않는다 —
  `globalPrevSha` 가 서비스의 직전 태그로 고정되기 때문이다.

### `HTTP request failed (network/timeout).` (Gemini refine)

* 대부분 **curl 타임아웃**이다. draft 가 크면 프롬프트가 약 150KB(≈40K 토큰)까지 가고,
  `max_output_tokens: 50000` 이라 응답까지 수 분이 걸린다.
* 에러 로그의 `max_time` / `retry` / `body bytes` 를 확인하고 `max_time` 을 올린다.
* 소요 시간이 `(1 + retry) * max_time` 에 정확히 맞으면 타임아웃이 확정이다
  (예: `max_time=60 retry=3` → 4분 03초).

### `Gemini response is incomplete: finishReason=MAX_TOKENS`

응답이 중간에 끊겼다는 뜻이다. 로그의 토큰 내역으로 원인이 갈린다.

```
finishReason=MAX_TOKENS | tokens: prompt=151877 output=3442 thoughts=46554 total=201873
  | maxOutputTokens=50000 thinkingBudget=<default>
```

* `thoughts` 가 예산 대부분을 먹었다면 → `thinking_budget` 을 지정해 사고를 묶는다
* `output` 이 예산을 다 쓰고도 안 끝났다면 → 출력 분량 자체가 많다.
  `compact_batch_size` 를 줄여 PASS 1 배치를 더 잘게 나눈다
* PASS 1 배치 하나가 잘리면 로그에 `batch batch_NNN.md: ...` 로 어느 배치인지 나온다

### 릴리즈 노트가 Notion 에서 계층 없이 평면으로 나온다

PASS 2 모델이 프롬프트의 4스페이스 지시 대신 입력(압축본, 2스페이스)을 따라
2스페이스로 내보낸 경우다. 파서가 출력의 최소 들여쓰기 폭을 감지해 흡수하므로
지금은 자동 처리되며, 감지값이 로그에 남는다:

```
release notes indent unit = 2 (observed widths: [2])
```
