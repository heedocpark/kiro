---
inclusion: fileMatch
fileMatchPattern: 'aidea-api-agent/**'
---

# AIDEA-API-AGENT

AnXer/aidea 플랫폼의 **에이전트 정의·버전관리·실행 서비스**. 에이전트 = 시스템 프롬프트 + MCP 툴 설정 + 모델/검증 설정(버전 단위 보관). 호출자가 `Messages`(텍스트)를 전달하면 SpinX를 통해 LLM을 실행하고 결과를 반환한다.

- **모듈 경로**: `github.com/mz-team-ax/aidea-api-agent`
- **Go 버전**: 1.26.3 (Alpine 3.23)
- **경로**: `/Users/phddy/Projects/dx/aidea-api-agent`
- **HTTP 포트**: `0.0.0.0:5005`

## 멀티 서비스 구성 내 위치

| 서비스 | 역할 | 포트 |
|--------|------|------|
| `aidea-api-workflow` | Argo 워크플로우 오케스트레이션 | 5004 |
| `aidea-api-agent` | 에이전트 API + CLI (본 서비스) | 5005 |
| `aidea-api-spinx` | LLM 게이트웨이 | - |
| `aidea-api-common` | 공용 라이브러리 | - |
| `hexim` | 공용 프레임워크 라이브러리 | - |

`go.work`가 `.`, `../aidea-api-common`, `../hexim` 를 함께 묶는다. 세 저장소는 로컬에서 같이 개발된다.

## 기술 스택

- **웹 프레임워크**: Fiber v3 (`github.com/gofiber/fiber/v3`)
- **DB/ORM**: MySQL + SQLBoiler v4 (`github.com/aarondl/sqlboiler/v4`), nullable는 `aarondl/null/v8`
- **CLI**: Cobra + Viper (`cmd/` 바이너리용)
- **로깅**: Uber zap (`go.uber.org/zap`) + 파일 로테이션
- **설정/env**: `yaml.v3`, `joho/godotenv`, `aidea-api-common/configs`
- **JSON**: `json-iterator/go` (list 뷰용 태그 기반 마샬링)
- **UUID**: `google/uuid` (UUID v7)
- LLM SDK는 직접 쓰지 않고 **모든 LLM 호출은 SpinX를 경유**한다.

## 3개 바이너리

Docker 이미지에 모두 빌드된다 (Dockerfile 참고).

- `aidea` (`main.go`) — Fiber HTTP API 서버
- `agent` (`cmd/agent`) — 에이전트 1회 실행 후 `POST /-/run`으로 결과 저장하는 Cobra CLI
- `workflow` (`cmd/workflow`) — Argo task Pod 내에서 단일 워크플로우 태스크 실행 CLI

## 디렉토리 구조

```
main.go          # HTTP 서버 진입점 (Fiber, :5005)
cmd/agent/       # agent 실행 CLI (cli.go, main.go, service.go)
cmd/workflow/    # workflow task CLI (Argo task 컨테이너)
router/
  index.go       # 최상위 Router(): /, /health, auth 미들웨어 + agent 라우트 연결
  agent/         # 핸들러: agent.go, run.go, version.go, subscribe.go
                 #   index.go (라우트 트리 + 서비스 init), type.go (기본 프롬프트 템플릿/상수)
service/agent/   # service.go — DB 접근 계층 (모든 쿼리 SQLBoiler)
agent/           # 실행 엔진: chat.go, runner.go, worker.go, verifier.go,
                 #   orchestrator.go, prompt.go, parse.go, history.go,
                 #   response_format.go, spinx.go, type.go, index.go (+ *_test.go)
types/           # 도메인/요청/응답 타입 + 변환 헬퍼 (uuid.go, context.go 등)
database/        # SQLBoiler 생성 코드 — 직접 수정 금지
middleware/      # exception.go — Fiber ErrorHandler()
config/          # index.go (Config 로더) + local.yaml, dev.yaml, live.yaml
```

## 실행 엔진 (`agent/` 패키지)

각 구성요소는 인터페이스 + `XxxConfig` + `NewXxx(c)` 생성자 패턴으로 되어 있다. 반드시 생성자로 조립한다: `NewWorker`/`NewVerifier`/`NewRunner`/`NewCoordinator`/`NewOrchestrator`/`NewChat`/`NewRecorder`. Config 필드가 zero면 `agent/index.go`의 기본값 사용 (`defaultMaxRetry=3`, `defaultMaxToken=5000`, `defaultMaxSubTasks=5`, `defaultMaxParallel=3`).

구성요소별 모든 LLM 호출은 `spinxCall(ctx, SpinxRequest{...})`(`spinx.go`)을 통해 **스트림(`llm.NewStream(true)`)으로** SpinX `chat` 엔드포인트를 친다. 응답/도구호출/usage/guardrail 이벤트를 모아 `(content, tools, usages, err)`를 반환한다.

### Worker (`worker.go`)
- `Run(ctx, in Input, history []Failure) (*Response, llm.Usages, error)` — Task를 1회 수행.
- SpinX 호출 → `parseWorkerOutput`으로 최종 JSON `{pass, result, reason, tools}` 파싱.
- **내부 재시도**(`MaxRetry`, 기본 3): 에러/파싱 실패/자체판단 실패(`pass:false`)를 Worker **안에서** 재시도한다. Runner의 턴을 소모하지 않는다.
- 응답 파싱 실패 시 그때까지의 도구 결과(`carried`)를 다음 시도에 넘겨 **재조회를 막는다**. 모델이 `tools`에 인용한 `ToolIds`만 결과로 필터링.
- 빈 응답(`errSpinxEmptyResponse`)은 `failures`에 쌓지 않고 `emptyRetryDelay`(2s) 후 재시도.
- 내부 재시도 소진 + 자체판단 실패면 마지막 `Response(Pass=false)`와 **nil 에러** 반환. 에러를 반환하는 경우는 guardrail 차단/ctx 취소/에러 재시도 소진뿐.

### Verifier (`verifier.go`)
- `Verify(ctx, in, result) (*Verdict, llm.Usages, error)` — Goal 충족 여부 판정. **도구를 쓰지 않는다**(MCP 없음).
- 응답 `{checks[], pass, reason}`을 `parseVerdict`로 파싱. `pass:true`인데 `checks`에 "불충족"이 있으면 형식 오류로 처리.
- `MaxRetry`(기본 3)만큼 파싱/에러 재시도.

### Runner (`runner.go`)
- `Run(ctx, in) (*RunResponse, llm.Usages, error)` — Worker(+선택적 Verifier)를 구동하는 **턴 루프**.
- Verifier가 nil이면 검증 생략. Runner의 재시도는 "진짜 재시도"(턴 소모)만 담당한다.
- 누적 `Failure` 이력을 `recent()`(최근 `maxHistory=3`건)로 추려 매 턴 Worker에 재주입.
- 검증 **에러**가 나면 Worker를 다시 돌리지 않고 같은 응답(`pending`)으로 검증만 재시도(턴은 소모).
- 모든 턴 소진 시 마지막 `Response`와 에러를 함께 반환. 한 번도 응답을 못 받았으면 nil+에러.
- `Failure.Source`: `error`(시스템) / `parsing error` / `self`(자체판단 실패) / `verifier`(불충족 판정).

### Orchestrator (`orchestrator.go`)
- `Plan(ctx, in) ([]SubTask, ...)` — 입력을 **서로 독립적인** 하위 작업(≤`MaxSubTasks`)으로 분할. `pass:false`면 계획 불가(재시도 안 함, `er.WithFatal()`).
- `Aggregate(ctx, in, results) (*Response, ...)` — 성공한 하위 작업 result만 근거로 최종 답 취합.
- 응답 형식 검증 실패 시 거부 사유(`fail_reasons`)를 붙여 재요청.

### Coordinator (`orchestrator.go`)
- `NewCoordinator`는 **Runner 인터페이스를 구현**하므로 호출부에서 Runner 대신 끼울 수 있다.
- 흐름: `Plan` → 하위 작업별 `NewRunner(sub)` 실행 → `Aggregate` → (선택) 최종 Verifier.
- **주의: 현재 병렬 실행이 꺼져 있다.** `execute()`에서 `g.SetLimit(c.cfg.MaxParallel)`가 주석 처리되고 `g.SetLimit(1)`로 하드코딩 → 사실상 순차 실행. (`MaxParallel` 설정은 무시됨)
- 하위 작업은 Task/Goal만 받는다(원본 요청/선행 결과 미전달). 일부 실패해도 성공분으로 취합 시도, 전부 실패면 에러. `runSub`는 panic recover.

### Chat (`chat.go`)
- `Stream(ctx, messages) iter.Seq2[*types.StreamResponse, error]` — Go 1.23+ iterator로 SSE 스트림.
- Task/Goal/검증 구조와 무관한 **대화형**. system 메시지는 호출부에서 제거 후 넘기고, Chat이 자체 system prompt를 선두에 붙인다.

### Recorder / history (`history.go`)
- **주체별 기록(`Record`) 트리** 구조 (`{"version":3,"records":[...]}`로 `agent_run.history`에 저장, `records`는 최상위 기록). 이전의 평면 이벤트 배열은 2계층 중첩에서 하위 작업 id(`s1`…)가 겹치고 한 배열에 69개씩 쌓여 읽기 어려워 바꿨다. `version` 없는 배열은 v1(평면 이벤트), `version:2`는 평면 기록.
  - `orchestrator` 기록(`Coordinator.Run`): 입력 + `steps`(`plan`/`aggregate`/`final_verify` 시도) + `children`(나눈 하위 작업의 기록). Plan의 **도구 호출과 분할 이유**도 남는다. `aggregate`는 별도 기록이 아니라 이 기록 안의 한 스텝이다.
  - `runner` 기록(`Runner.Run`): 입력 + `steps[]`(Worker 시도 `worker`와 검증 `verifier`가 **같은 레벨에 시간순**, `turn` 번호로 짝을 구분). orchestrator의 `steps`와 같은 이름·모양. 객체는 MySQL JSON 컬럼이 키를 재정렬(짧은 키 먼저)해 순서가 뒤바뀌므로 흐름은 배열로 둔다. "첫 turn·첫 attempt에 통과"는 `steps`에서 `turn==1`인 `worker` 1건 + `verifier` pass로 판별.
  - 기록 `id`는 경로(`run`, `run/s1`, `run/s1/s2`). `beginRecord(ctx, role, name, in)`이 부모 기록의 id + ctx의 하위 작업 id로 만들고, `coordinator.runSub`가 `withSubtask`로 하위 작업 id를 넣는다. 부모-자식은 중첩(`children`)으로 표현해 `parent_id`는 없다.
  - 기록은 주체가 끝날 때 `defer`로 완성해 부모의 `children`에 붙고(`recScope.attach`), 최상위만 `Recorder.Add`(에러·취소로 끝나도 남음). `seq`는 시작 순서이고, `children`은 **계획 순서**로 정렬(`childLess`, 병렬 실행이면 끝나는 순서가 제각각이라 `coordinator.runSub`가 넘기는 `order`를 우선 쓴다). 통과한 기록의 `reason`은 응답의 **타협 사항**(pass여도 목표를 온전히 채웠는지 구분). `elapsed_ms`는 기록별 소요 시간(절대 시각은 안 남김).
- 호출부가 `NewRecorder()` 생성 후 `WithRecorder(ctx, rec)`로 ctx에 주입. **ctx에 Recorder가 없으면 `beginRecord`/`record(...)`는 no-op** (테스트/비기록 경로 안전). 스트림(Chat) 경로는 기록하지 않는다.
- 크기 상한(`maxRecordBytes=2MB`)은 하위 기록이 부모에 붙을 때 자기 크기를 `reserve`해 지킨다. 초과 시 최상위 `truncated:true` 표시 후 넘치는 하위 기록만 버리고 최상위 기록은 유지. 긴 텍스트는 `clip()`(멀티바이트 안전)으로 자른다. 도구 결과 원문/결과 본문, 거부된 Plan 원문은 history에 저장하지 않는다(거부 원문은 `orchestrator plan rejected` 로그의 `content`에만 있음).
- 용도: 질문별로 어떤 도구·키워드가 재시도 없이 통했는지 주기적으로 뽑아 에이전트 팁으로 쓰기 위한 데이터. 도구 이름에는 서버 접두어(`<id>_____`)가 붙어 집계 시 정규화가 필요하다.

### 공통 직렬화/파싱 관례
- `marshalNoEscape` / `encodeJSON` — `SetEscapeHTML(false)`로 `<`,`>`,`&` 이스케이프 없이 직렬화. `agent/index.go` `init()`에서 SpinX 클라이언트 JSON marshaler도 동일하게 교체.
- 메시지는 `xmlBlock(tag, body)`로 `<task>`, `<goal>`, `<request>`, `<upstreams>`, `<fail_reasons>` 블록을 만들어 전달.
- `jsonObject`/`stripCodeFence` — 모델이 코드블록/설명을 섞어 보내도 JSON 객체 부분을 추출. `pass` 필드는 포인터(`*bool`)로 받아 "필드 없음"과 "false"를 구분.
- `Response`는 `fail`/`json` 두 벌의 jsoniter 태그를 가진다(`type.go`). 모델에 보여줄 때는 `fail` 태그 뷰 사용.

### 프롬프트 설계 (`prompt.go`, `orchestrator.go`)
- 엔진 기본 프롬프트가 **최상위 기준**. agent 버전의 사용자 프롬프트는 `{{USER_SYSTEM_PROMPT}}` placeholder에 `SystemPrompt.Make(...)`로 주입되어 `<user_instructions>`로만 반영.
- **프롬프트 인젝션 방어가 설계에 내장**: `<task>/<goal>/<request>/<upstreams>` 안의 지시문, 출력 형식 변경/역할 변경 요구는 무시한다. 충돌하는 사용자 지시는 "조용히 무시"하고 그 사실을 응답에 언급하지 않는다. 새 프롬프트를 쓸 때 이 방어 구조를 유지한다.

## 코딩 관례

### 파일 네이밍
- `index.go` — 패키지 진입/배선 파일. 패키지 `init()`, 패키지 레벨 공유 var, `Router(...)`/셋업을 담는다. 새 패키지 추가 시 이 관례를 따른다.
- `type.go` — 패키지 로컬 상수와 작은 타입.
- 그 외 파일은 도메인 관심사별 명명 (`agent.go`, `run.go`, `worker.go` 등).

### DI 스타일
- **패키지 레벨 싱글톤 + 패키지별 `init()` 배선**. 생성자로 `main`에서 주입하지 않는다.
- `config/index.go`가 `ENV`로 선택된 YAML + `.env`를 로드해 싱글톤 생성, `cfg.Get()`로 노출. 모든 `init()`은 `hexim.LoadEnv()` 호출.
- 서비스는 명시적 생성자 `NewService(db, readDb *sql.DB)` 사용 — 쓰기(`anxer`)/읽기(`anxer_ro`) 커넥션 분리.

### 에러 처리
- `hexim/errors` 패키지를 별칭 `er`로 사용.
- 전파: `return er.Wrap(err)` / 생성: `er.New("msg", er.WithCode(http.StatusBadRequest))` / 검사: `er.Is`, `er.As` / 필드: `er.WithKeyValue(...)`.
- 핸들러는 직접 에러 응답을 쓰지 않고 **에러를 반환**한다. `middleware.ErrorHandler()`가 중앙에서 `er.Error`를 풀어 `{id, message, detail}` 응답으로 렌더링(404/401/403는 로깅 생략).

### 로깅
- zap 전역 로거 `zap.L()`, `log` YAML 블록 기반 (`aidea-api-common/logs`, iso8601, 일 단위 로테이션).

### Fiber v3 핸들러
- 컨텍스트 값: `c.Value(types.ContextKeyXxx)` 후 타입 단언.
- 바디 바인딩: `c.Bind().JSON(&x)`.
- 응답: `c.JSON(x)`, 빈 리스트는 `http.StatusNoContent`(204).
- 리스트 뷰는 `list` 구조체 태그 기반 전용 jsoniter 설정(`listJson`)으로 마샬링해 태그된 필드만 노출.
- 컨텍스트 키는 `types/context.go`에 문자열 상수로 집중 관리.

### 테스트
- 표준 `testing` + `stretchr/testify`. 테스트는 `agent/` 패키지 내 코드 옆에 위치.
- `*_Live` 네이밍 테스트는 실제 SpinX/MCP를 호출 → 의도적으로만 실행, 일반 CI에서 돌리지 않는다.

## 데이터베이스 / ORM

- **SQLBoiler v4**, 설정은 `sqlboiler.toml` (whitelist: `agent`, `agent_version`, `agent_run`, `agent_subscribe`, `agent_usage`, `user`). 생성 코드는 `database/`에 있으며 **직접 수정 금지**, 스키마 변경 시 재생성하고 `README_DB.md` 동기화.
- **ID**: UUID v7 → `BINARY(16)` 저장. 앱이 생성(`types.NewUUID()`). SQLBoiler 모델은 `[]byte`, 쿼리 시 `id[:]`, 복원은 `types.ToUUID([]byte)`. nullable UUID는 `null.Bytes`.
- **쿼리 스타일**: 타입드 query mod — `database.Agents(database.AgentWhere.ID.EQ(id[:]), ...)`, `qm.Select(...)`, `qm.OrderBy(...)`, 종단 메서드 `.One(ctx, s.readDb)` / `.All(...)` / `.Count(...)`.
- **읽기/쓰기 분리**: 읽기는 `anxer_ro`, 쓰기는 `anxer`. 쓰기 직후 복제 지연 주의.
- **부분 업데이트**: SQLBoiler whitelist — `ag.Update(ctx, s.db, agent.WhiteList())`. insert는 `boil.Infer()`.
- `version_no`는 트랜잭션 내 `MAX(version_no)+1` (아직 `SELECT ... FOR UPDATE` 행 잠금 없음 — README_DB §4-2 참고).

## 요청 흐름

`router/index.go` → `router/agent/index.go` 라우트 트리 → `router/agent/*.go` 핸들러 → `service/agent`(DB) 및/또는 `agent/`(실행 엔진) → `types.ToXxx` 변환 → `c.JSON`. 에러는 `er.Wrap`/`er.New`로 버블업되어 `middleware.ErrorHandler()`가 렌더링.

- **인증**: `/`, `/health` 제외 모든 라우트가 `middlewares.Authentication(...)`(JWT + Redis 세션, `AIDEA_ACCESS_ROLE`로 역할 필터) + `middlewares.RequestIdGenerator()` 통과. 사용자: `c.Value(types.ContextKeyUser).(*typ.User)`.
- **프롬프트 조합**: 엔진 기본 프롬프트가 우선. 버전의 `system_prompt`를 `{{AGENT_SYSTEM_PROMPT}}` placeholder에 주입(`systemPrompt.Make()`). 충돌하는 사용자 지시는 무시.
- **Run 저장**: 앱 생성 UUID로 `types.AgentRun` 구성 → 엔진 실행 → `agentService.SetRun(...)`(트랜잭션으로 `agent_run` + 모델별 `agent_usage` insert). `history`는 DB에만 저장, 응답에서는 제거(`run.History = nil`).
- **SSE 스트리밍**: `text/event-stream` 헤더, `c.SendStreamWriter`, `data: {json}\n\n` 프레임, 5분 타임아웃, panic recover. 첫 이벤트 `id`를 run UUID로 교체. 스트림 모드는 검증 없음.

## 세 진입점의 엔진 조립 비교

세 바이너리 모두 `agent/` 엔진을 쓰지만 **조립 방식과 전달하는 입력이 다르다.** 특히 `Input`의 `Upstreams`/`RunInput` 전달 여부가 다르다.

| 항목 | API `invoke` (`router/agent/run.go`) | agent CLI (`cmd/agent/service.go`) | workflow CLI (`cmd/workflow/service.go`) |
|---|---|---|---|
| 조립 | `Orchestrator` 플래그면 Coordinator, 아니면 단일 Runner | 단일 Runner 고정 | **Coordinator 고정** |
| `Input.Upstreams` | ✗ 미전달 | ✗ 미전달 | ✓ `Load()`로 선행 task 결과 |
| `Input.RunInput` | ✗ 미전달 (`Input{Task, Goal}`만) | ✗ 미전달 | ✓ `s.run.UserInput` |
| Orchestrator Model | `version.Model` | — | **하드코딩 `anthropic/claude-haiku-5-5`** |
| Orchestrator MaxToken / MaxSubTasks | 10000 / 3 | — | 10000 / 3 |
| Coordinator 최종 Verifier | 주석 처리(없음) | — | 주석 처리(없음) |
| 결과 저장 | `agentService.SetRun` 직접 DB | `agent` API `/-/run` 호출 | `agent` API `/-/run` + output 파일 |

## 워크플로우 Task 간 데이터 전달 (호텔→여행계획 류)

워크플로우에서 에이전트는 task로 쓰이고, 선행 task 결과가 후행 task의 `upstream`으로 주입된다. 흐름:

1. `cmd/workflow/service.go:Run`이 `Load()`로 의존 task 결과 파일(`run-{runID}-task-{depID}.json`)을 읽어 `[]agent.Upstream` 구성 → `Input{RunInput, Task, Goal, Upstreams}`로 Coordinator에 전달. (여기까지 upstream 정상)
2. `Orchestrator.Plan`은 `<upstreams>`를 **본다**(`planMessages`→`baseMessages`). 하지만 Plan은 "subtask의 `task`/`goal` 문자열에 upstream 핵심값을 직접 적어라"에만 의존한다.
3. **소실 지점 — `coordinator.runSub`가 서브태스크에 `Input{Task: sub.Task, Goal: sub.Goal}`만 넘긴다.** `Upstreams`/`RunInput`을 전달하지 않으므로, 서브태스크 Worker는 Plan이 문자열에 적어넣은 요약본만 본다. 선행 결과가 길면 Plan 모델(하드코딩 haiku, MaxToken 10000)이 요약/누락하고 그 손실분이 그대로 Worker로 전달된다.

→ "선행 결과 S3 업로드 → upstream 주입"까지는 정상이고, **Coordinator가 단건을 Plan→subtask로 분할하면서 subtask 경계에서 upstream 원본이 끊기는 것**이 핵심 실패 원인. Worker/Verifier 프롬프트는 이미 `<upstreams>`/`<request>`를 처리하도록 되어 있어, `runSub`가 부모 `Input`의 `Upstreams`/`RunInput`을 그대로 넘기면 해소 가능하다(다만 "서브태스크는 독립적" 설계 철학과의 절충 필요).

## 빌드 / 실행 / 테스트

```bash
# API 서버 로컬 실행 (ENV=local 기본, config/local.yaml + .env 로드)
cd /Users/phddy/Projects/dx/aidea-api-agent && go run .

# 3개 바이너리 빌드 (Dockerfile과 동일)
go build -o bin/aidea main.go
go build -o bin/agent ./cmd/agent
go build -o bin/workflow ./cmd/workflow

# 운영 빌드 태그
go build -tags production ...

# 테스트 ( _Live 테스트는 외부 서비스 호출 → 의도적으로만 )
go test ./...
```

- **ENV**로 config 파일 선택. 필수 env: `AIDEA_ACCESS_ROLE`(비어 있으면 panic), `AGENT_SPINX_KEY`, `MYSQL_USER`, `MYSQL_PASSWORD`, `JWT_SECRET_KEY`, `APP_NAME`, `POD_IP`. 루트 `.env` 자동 로드.
- **Docker**: 멀티스테이지·멀티아치(amd64/arm64), 베이스 이미지는 Harbor(`ax-harbor.cjmezzomedia.com`). `dumb-init` 엔트리포인트(Fiber prefork는 PID>1 필요).
- **CI/CD (Jenkinsfile)**: 병렬 아치 빌드 → Harbor push → 멀티아치 manifest → `argocd-apps` 리포 `kustomize edit set image`로 GitOps 배포. 이미지 해시 미변경 시 push/deploy 생략.

## 알려진 개선 여지 / 버그 (코드 확인)

> 아래 1~4번은 **커밋 당시 기준**. `cmd/workflow/service.go`는 아래 "2계층 오케스트레이션 재설계"에서 상당 부분 바뀌었으니 그 섹션을 우선 참고. (`router/agent/run.go`는 아직 단층이라 여전히 해당)

1. ~~**Coordinator가 서브태스크에 upstream/run_input 미전달**~~ → 재설계에서 `runSub`가 상위 Upstreams+RunInput 전달하도록 해결(workflow 한정, API invoke는 미해결).
2. **Orchestrator 모델/파라미터 하드코딩** — `orchestratorModel`, `MaxToken:10000`, `MaxSubTasks`. 버전 설정으로 받는 게 숙제(미해결).
3. ~~**Coordinator 병렬 비활성(`SetLimit(1)`)**~~ → 재설계에서 `Parallel` 플래그로 대체(fanout은 병렬, stage는 순차).
4. **Coordinator 최종 Verifier 주석 처리** — 여전히 세 진입점 비활성(`//Verifier: ...`).
5. **스트림 모드가 `model` 대신 `verifier_model` 사용** — `router/agent/run.go`의 `NewChat(... Model: version.VerifierModel ...)`.
6. `DELETE /agent/{id}`의 `remove()`, `deleteVersion()`이 nil 반환 no-op 스텁.
7. **workflow/agent CLI 에러에 token이 로깅될 수 있음** — `er.WithKeyValue("body", res.Body())` 등으로 응답 본문이 필드에 실림. go-conventions의 "토큰·시크릿 로그 금지"와 충돌.
8. `version_no` 발급 시 `SELECT ... FOR UPDATE` 행 잠금 부재(동시성 gap, README_DB §4-2).
9. `sqlboiler.toml`에 운영 DB 자격증명이 평문 — 환경변수 분리 권장.

---

## 2계층 오케스트레이션 재설계 (진행 중, 토큰/품질 최적화)

여행 워크플로우(호텔 조사 → 비교·추천)에서 **단일 Worker가 도구를 과다 호출해 입력 컨텍스트가 모델 윈도우를 초과**하는 문제를 풀기 위해, Coordinator를 **2계층 중첩 구조**로 재설계했다. (`cmd/workflow/service.go`에 적용)

### 구조
```
바깥 Coordinator (Name: "stage") — 순차 단계
  Orchestrator: DefaultPlanSystemPrompt (도구 X)
  Parallel: false, PipeUpstream: true     ← 발굴 → 조사 → 비교·추천, 앞 단계 결과를 다음 단계 Upstreams 로 전달
  NewRunner → 안쪽 Coordinator (Name: "fanout") — 대상별 병렬
                Orchestrator: DefaultWorkerPlanSystemPrompt + MCP(검색 도구)
                Parallel: true             ← "호텔A 조사 / 호텔B 조사" 동시 실행
                NewRunner → 일반 Worker+Verifier Runner (호텔 1곳만 조사 → 토큰 분산)
```
- **두 Plan은 역할이 다르다.** 바깥(Orchestrator Plan)은 순차 단계 골격만(도구 없음), 안쪽(Worker Plan)은 검색으로 대상 파악 후 대상별 병렬 분할(도구 있음).
- `NewCoordinator`가 `Runner` 인터페이스를 구현하므로 `NewRunner` 팩토리에 중첩 Coordinator를 끼우는 조립만으로 2계층이 된다. 재귀가 아니라 조립으로 깊이를 고정해 무한 루프 걱정이 없다(depth 카운터 불필요).

### 추가된 구성요소 (`agent/orchestrator.go`)
- `CoordinatorConfig.Name` — 로그 식별자(`coordinator=stage`/`fanout`), 하위 로그에 전파.
- `CoordinatorConfig.Parallel` — true면 `executeParallel`(동시), false면 `executeSequential`(순차).
- `CoordinatorConfig.PipeUpstream` — 순차일 때 앞 단계 성공 결과를 뒤 단계 `Upstreams`로 누적 전달(파이프라인). false면 결과 안 넘기고 독립 순차.
- `OrchestratorConfig.MCP` — Plan이 대상 파악에 검색 도구를 쓸 수 있게. `request()`가 실어 보냄.
- `SubTask` 1개면 Aggregate/최종 Verifier 생략(이미 검증 완료).
- `runSub`가 상위 `in.Upstreams`(워크플로우 선행 task) + 내부 앞 단계 결과(`ups`)를 **합쳐서** 하위에 전달. `RunInput`도 전달(Plan 뭉갠 디테일 보완).
- `plan created` 로그 — Plan이 만든 subtask(id/task/goal)를 요약 출력(`planSummary`).

### 토큰/품질 디버깅에서 얻은 핵심 교훈 (실전 Live 로그 기반)

1. **tool_use + response_format(strict JSON)은 충돌한다.** 도구를 쓰는 LLM 호출에 strict JSON 스키마를 걸면, 모델이 멀티턴 중 형식을 맞추려 **필드를 빈 값으로** 채운다.
   - Worker는 이 이유로 response_format을 뺐다(커밋 `9fd8ad9`, `//ResponseFormat: workerResponseFormat()`).
   - **fanout Plan(도구 사용)도 같은 문제** → goal이 `""`로 반복 생성되어 `validatePlan`이 거부 → 재시도 소진 → 전체 실패. Plan이 sonnet인데도 발생(= 모델 역량 아님, 구조 문제).
   - **해결**: `planRequest`가 `usesTools()`(MCP 활성)면 response_format 생략. 도구 없는 stage Plan은 유지. 안전망은 프롬프트 출력 안내 + `parsePlan`/`validatePlan`.

2. **Worker가 pass:false면 Verifier를 안 거친다.** 그래서 "Verifier가 타협 가이드를 준다"는 개선은 **Worker가 pass:true로 올려줄 때만** 작동. haiku Worker가 "확정가(미래 날짜 실시간가) 못 찾음"을 스스로 실패로 판정해 재시도를 소진(62K 토큰)하던 문제 → **Worker 프롬프트의 Goal 타협 섹션**에 "검색으로 못 얻는 값은 즉시 '확인 안 됨' 타협 + pass:true. 핵심 항목 미확인만으로 pass:false 하지 마라. 그 판단은 Verifier가 한다"를 명시.

3. **검증은 결과를 보고 과정은 보지 않는다.** Verifier/Plan이 "출처 N곳 이상 시도" 같은 **절차 요구**를 검증 조건으로 삼아 reject → 재시도 폭증. → Verifier 프롬프트에 "절차 요구는 검증 대상 아님, 결과물만 판정" 명시, Plan goal 작성 규칙에 "절차는 task에만, goal엔 결과 조건만" 명시.

4. **역할별 모델 분리.** Plan·Verifier = sonnet, 단순 조사 Worker = haiku(DB `version.MaxTokens`/모델 설정). 조사 Worker가 호출 수가 가장 많아 haiku로 비용 절감. 단 haiku는 JSON 형식을 자주 깨뜨림(`invalid character 'ì'`/`'\n'`).

5. **JSON 파서 관용성 강화** (`parse.go:jsonObject`). response_format 못 쓰는 Worker가 깨진 JSON을 뱉는 걸 구제: 첫 `{`부터 **중괄호 균형**이 맞는 `}`까지만 추출(앞뒤 설명 제거, 중첩 중괄호 정확히), **문자열 리터럴 안 생 제어문자**(줄바꿈 등)를 이스케이프. 세 파서(Worker/Aggregate/Verdict) 모두 거친다.

6. **`validatePlan`은 빈 task/goal을 모아서 한 번에 거부** — 재시도 때 모델이 전부 한 번에 고치도록. (단, 1번처럼 response_format이 원인이면 재시도로도 안 고쳐짐 → response_format 제거가 근본 해법)

### 로그 규약 (`spinx.go`, `orchestrator.go`)
- `spinx send request`/중복 제거. 호출 1건당 `spinx done`에 `spinx_id`, `input_bytes`, `tools`, `input_tokens`, `output_tokens` 요약(비용 추적).
- `coordinator` 필드로 레이어 구분(`stage`/`fanout`). `plan created`로 계획 가시화.
- 참고: `worker/runner/verifier/coordinator`의 start/done timer 로그는 `hexim.NewTimer`(공용) 소관이라 agent에서 못 끔.

### 남은 과제
- fanout response_format 제거 효과 Live 재확인(goal 빈 값 해소 여부).
- 이중 Aggregate(fanout Aggregate + stage Aggregate) 중복 — 비싼 호출(input 40K~50K)이라 제거 검토.
- Verifier가 `pass:true`일 때 reason을 "placeholder"/빈 값으로 뱉는 경향(sonnet) — 판정은 맞으나 reason 부실.
- 모델/파라미터 하드코딩(`orchestratorModel`, MaxToken/MaxSubTasks) — 버전 설정으로 받기.

> 주의: 이 재설계는 `cmd/workflow/service.go`에만 적용됨. `router/agent/run.go`(API invoke)는 아직 단층 구조. 작업은 커밋 전 git stage 단위로 사용자가 관리 중.
