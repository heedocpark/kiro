---
inclusion: fileMatch
fileMatchPattern: 'aidea-mcp/agent-mcp-server/**'
---

# AGENT-MCP-SERVER

LLM 클라이언트가 **agent와 workflow를 직접 만들고 조회**할 수 있게 도구(MCP tool)로 노출하는 서버. 자체 DB나 비즈니스 로직 없이 `aidea-api-agent`(:5005)와 `aidea-api-workflow`(:5004)의 REST API를 호출하는 **얇은 프록시**다. 최종 목표는 "목표 입력 → agent 생성 → 버전 생성·선택 → workflow 생성 → task(DAG) 추가"를 LLM이 도구 조합만으로 수행하는 것이다. 작업 분해와 agent 선택 판단은 MCP를 쓰는 LLM이 하고, 이 서버는 저수준 CRUD 도구만 제공한다.

- **모듈 경로**: `github.com/mz-team-ax/aidea-mcp/agent-mcp-server`
- **Go 버전**: 1.26.3
- **경로**: `/Users/phddy/Projects/dx/aidea-mcp/agent-mcp-server` (이전 이름 `workflow-mcp-server`에서 변경됨)
- **포트/엔드포인트**: `0.0.0.0:5040`, MCP 엔드포인트 `/agent`, 서버 이름 `agent-server`, 헬스체크 `GET /health`
- `go.work`가 `.`, `../../aidea-api-common`, `../../hexim`을 묶는다. 같은 `aidea-mcp` 저장소의 다른 서버(`admin`, `search`, `kiwi`)와 독립된 모듈이다.

> 백엔드 동작은 `aidea-api-agent.md`, `aidea-api-workflow.md`를 본다. 공통 Go 규칙은 `go-conventions.md`, MCP 서버 프레임워크(`hexim/mcp`)는 `hexim.md`를 본다. 여기서는 이 서버에 고유한 구조와 패턴만 다룬다.

## 기술 스택

- **MCP**: `hexim/mcp` (Fiber 위에 `modelcontextprotocol/go-sdk`를 얹은 래퍼). Streamable HTTP, `Stateless: true`.
- **도구 스키마**: `github.com/google/jsonschema-go/jsonschema`
- **백엔드 호출**: `aidea-api-common/types`의 `ApiConfig`/`ApiRequest`(resty 래퍼). 엔드포인트는 yaml로 선언한다.
- **설정**: `ENV`(기본 `local`)로 `config/{local,dev,live}.yaml` 선택. 루트 `.env` 자동 로드.
- **인증**: `middlewares.Authentication(config.Jwt, redis, RoleAdmin, RoleApi, RoleUser)`로 들어오는 요청의 JWT를 검증한다.

## 인증 토큰 전달 (중요)

MCP 서버는 JWT를 **검증만** 하고, 백엔드 호출에는 **받은 `Authorization` 헤더를 그대로 전달**한다(`call()`이 `request.Extra.Header`에서 꺼내 설정). 그래서 권한 판단과 사용자별 데이터 필터는 백엔드가 한다. 서비스 계정 토큰을 따로 쓰지 않는다. 새 호출 경로를 만들 때도 `call()`을 거쳐야 이 전달이 보장된다.

## 디렉토리 구조

```
main.go                  # 서버 부트스트랩 + 도구 등록 (등록 목록의 기준)
config/
  index.go               # Config 로더 (ENV로 yaml 선택)
  local|dev|live.yaml    # api.agent / api.workflow 엔드포인트 정의 + redis/jwt 등
tool/
  call.go                # 공통 헬퍼: bind, call, listResult
  agent.go               # agent 도구 정의(Definition) — 이름/설명/입력 스키마
  agent_handler.go       # agent 도구 핸들러 (AgentHandler)
  workflow.go            # workflow, run 도구 정의
  task.go                # task 도구 정의
  workflow_handler.go    # workflow/task/run 도구 핸들러 (WorkflowHandler)
types/
  agent.go               # Agent, AgentVersion, AgentRun
  workflow.go            # Workflow, WorkflowTask, WorkflowRun, WorkflowRunTask
Dockerfile, Jenkinsfile  # 멀티스테이지 빌드(main.go → /usr/bin/agent), dumb-init
```

도구는 **정의(`XxxDefinition()`)와 핸들러(`(h *Handler) Xxx`)를 파일로 나눠** 둔다. 정의는 LLM이 읽는 계약이고, 핸들러는 백엔드 호출이다.

## 등록된 도구 (`main.go`가 기준)

| 영역 | 등록됨 | 미등록 (스텁/주석) |
|---|---|---|
| workflow | `workflow_list`, `workflow_get`, `workflow_save` | `workflow_delete` |
| task | `task_list`, `task_get`, `task_save` | - |
| run | `run_list`, `run_get`, `run_task_list` | `run_create`(실행), `run_task_get` |
| agent | `agent_list`, `agent_get`, `agent_save` | `agent_run`(실행), `agent_subscribe*` |
| agent version | `agent_version_list`, `agent_version_get`, `agent_version_save` | - |
| agent run | `agent_run_list`, `agent_run_get` | - |

**실행(`run_create`, `agent_run`)과 삭제(`workflow_delete`)는 의도적으로 보류**한 상태다. 핸들러는 `return nil, nil` 스텁이고 등록은 주석 처리돼 있다. 스텁을 등록하면 호출 시 문제가 되므로 구현 전에는 등록하지 않는다. 실행은 Argo 리소스를 쓰는 동작이라 구현 시 "사용자 요청 없이 호출하지 않는다"는 안내를 정의 설명에 유지한다.

## 핸들러 패턴 (모든 도구가 따른다)

```go
func (a *AgentHandler) AgentGet(ctx context.Context, request *mcp.CallToolRequest) (*mcp.CallToolResult, error) {
	in, err := bind[typ.Agent](request)                       // 1. 인자 파싱
	if err != nil { return nil, err }

	if in.ID == "" {                                          // 2. 필수값 검사 (백엔드 호출 전에 400)
		return nil, er.New("agent_id is required", er.WithCode(http.StatusBadRequest))
	}

	agent, err := call[typ.Agent](ctx, a.agentAPIConfig, request, "agent_get", func(r *types.ApiRequest) {
		r.SetPathParam("agent_id", in.ID)                     // 3. 호출 직전에 path/query/body 지정
	})
	if err != nil { return nil, err }

	return listResult(agent, "not found agent")               // 4. 결과 변환
}
```

공통 헬퍼(`tool/call.go`):
- `bind[T]` — 도구 인자를 `T`로 파싱. 인자가 생략/`null`이어도 zero value 반환(필수값은 핸들러가 검사).
- `call[T]` — `api.R(endpoint)`로 요청 생성 → `setup` 콜백 → `Authorization` 전달 → 실행 → 응답을 `T`로 디코딩. **200, 204를 성공**으로 보며(204는 `T`의 zero value), 그 외는 응답 본문을 메시지로, 상태 코드를 `er.Code`로 담아 에러 반환.
- `listResult(v, empty)` — 슬라이스는 항목마다 JSON 텍스트 1건, 구조체/맵은 JSON 텍스트 1건, nil/빈 슬라이스는 `empty` 메시지 1건, 그 외 타입은 에러.

**입력 구조체를 태그/리플렉션으로 자동 반영하지 않는다.** path/query/body는 핸들러가 `setup` 콜백에서 직접 지정한다(복잡도를 낮추려는 의도적 결정).

## 도구 입력 규칙

- 도구 정의의 **속성 이름 = 입력 구조체의 JSON 태그**. 불일치하면 값이 조용히 비어 필수값 에러가 난다.
- 대상 엔티티 자신의 id는 `id`, 부모/상위 id는 `agent_id`, `workflow_id`, `run_id`로 쓴다. (`agent_get`은 `id`, `agent_version_get`은 `id`+`agent_id`, `run_get`은 `id`+`workflow_id`.)
- `setup`의 `SetPathParam` 키는 yaml 경로의 `{...}` 이름과 같아야 한다(예: `{agent_id}`, `{version_id}`, `{run_id}`).
- 엔드포인트 이름은 yaml의 `api.<서비스>.endpoint` 키와 같다(`agent_list`, `agent_create`, `agent_update`, `task_update` …).

## 저장(Save) 도구 규칙

- `id`가 없으면 `*_create`, 있으면 `*_update` 엔드포인트를 쓴다(`hexim.Ternary`). 둘 다 `POST`다.
- **body는 바인딩한 입력 구조체 그대로(`r.SetBody(in)`)** 보낸다. 그래서 선택 필드에는 `omitempty`가 필수다.
  - 백엔드 수정 API는 기존 엔티티를 읽은 뒤 body를 **덮어쓴다**. 보내지 않은 필드는 유지되어야 한다.
  - UUID 필드에 `""`가 가면 백엔드 파싱이 400으로 실패한다. 선택 UUID 필드(`selected_version_id` 등)는 반드시 `omitempty`.
  - `[]byte`/`json.RawMessage` 필드는 `omitempty`가 없으면 `null`이 실려 기존 값을 지울 수 있다(`dependencies`가 그 사례).
  - 0/false가 의미 있는 값이면 `omitempty`가 빼 버리므로 포인터 타입을 쓴다. 현재 저장 도구에는 해당 필드가 없다.
- 응답은 저장된 엔티티이며 `listResult(entity, "")`로 반환한다.

## 페이징

`run_list`, `agent_run_list`만 페이징된다(`page.Result[T]` 응답, 백엔드 기본값 `page=1`, `per_page=10`). 핸들러는 **값이 0보다 클 때만** 쿼리로 보낸다. 0을 보내면 백엔드 기본값을 덮어써 빈 결과가 나온다. 정의에서도 `page`, `per_page`는 Required가 아니다. `run_task_list`는 페이징이 없다.

## 도메인 규칙 (도구 설명에 반영돼야 하는 백엔드 제약)

- **agent 실행 조건**: `status`가 `active`이고 `selected_version_id`가 지정돼 있어야 한다. 버전을 저장해도 자동 선택되지 않는다.
- **agent 설명은 버전에 있다**: agent 테이블에는 설명 컬럼이 없고 선택된 버전의 `description`이 agent 설명으로 파생된다. `agent_save`로는 설명을 저장할 수 없다.
- **task 필수 항목**: `agent_id`(생성·수정 모두). `dependencies`는 선행 task의 **UUID 배열**이며, 선행 task가 먼저 생성돼 있어야 한다(DAG).
- **자동 생성 순서**: `agent_save` → `agent_version_save` → `agent_save`(`selected_version_id`, `status: active`) → `workflow_save` → `task_save` 반복(선행 task부터).
- run `status`: `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `UNKNOWN`. `SUCCEEDED`/`FAILED`면 종료.

## 새 도구 추가 절차

1. 호출할 백엔드 엔드포인트를 `config/local.yaml`, `dev.yaml`, `live.yaml`의 `api.agent` 또는 `api.workflow`에 추가(`path`, `method`). 세 파일 모두 갱신한다.
2. `tool/*.go`에 `XxxDefinition()` 추가. 이름(snake_case), 한국어 설명(언제 쓰는지, 반환값, 선행 조건), `InputSchema`(속성, `Required`).
3. `types/`에 입력/응답 구조체 추가 또는 확장. JSON 태그는 정의의 속성 이름과 일치시키고, 선택 필드는 `omitempty`.
4. 핸들러 구현: `bind` → 필수값 검사 → `call` → `listResult`. 같은 패턴을 유지한다.
5. `main.go`에 `server.AddTool(...)` 등록.
6. 검증: `go build ./... && go vet ./... && gofmt -l .`

## 빌드 / 실행

```bash
cd /Users/phddy/Projects/dx/aidea-mcp/agent-mcp-server
go build ./...        # 빌드 검증
go vet ./...
gofmt -l .
go run main.go        # ENV=local 기본, config/local.yaml + .env
```

- 로컬은 `api.agent`가 `localhost:5005`, `api.workflow`가 `localhost:5004`를 가리킨다. 두 백엔드가 떠 있고 Redis/JWT 설정이 되어 있어야 호출이 성공한다.
- `PROCESS` env가 1보다 크면 Fiber prefork로 동작하며 도구는 자식 프로세스에서만 등록된다(`server.IsChild() || !preFork`).
- dev/live yaml은 클러스터 내부 DNS(`aidea-api-agent[-dev]....svc.cluster.local:5005` 등)를 쓴다. **로컬에서만 동작을 확인했고 dev/live 배포는 검증하지 않았다.**
- Docker: 멀티스테이지, `main.go`를 `/usr/bin/agent`로 빌드, `dumb-init` 엔트리포인트. Jenkinsfile 존재.
- 이 서버에는 테스트가 없다. 변경 검증은 빌드/vet과 로컬 수동 호출로 한다.

## 알려진 한계 / 남은 과제

- 핸들러가 에러를 `error`로 반환하면 JSON-RPC 프로토콜 에러가 되어 LLM이 백엔드 상태 코드를 도구 결과로 받지 못한다. 필요하면 `call` 경계에서 `CallToolResult{IsError: true}`로 바꾸는 정책을 한 곳에서 정한다(미결정).
- `types.Workflow`의 `user_no` 태그가 백엔드 응답 키 `userNo`와 달라 응답에서 값이 비어 나온다.
- `workflow_delete` 정의만 입력 키가 `workflow_id`이고 나머지 단건 도구는 `id`다. 구현 시 `bind[Workflow]`와 맞추도록 정렬해야 한다.
- 백엔드 쪽 갭이 MCP를 통해 그대로 노출된다: `workflow_list`/`getRunList` 등이 `userNo` 필터가 주석 처리돼 있어 전체 사용자 데이터가 조회될 수 있다(`aidea-api-workflow.md` 참고).
- `agent_run` 같은 동기 장시간 호출은 `ApiConfig`의 read 타임아웃(`timeout.read`)에 걸릴 수 있어 구현 시 엔드포인트별 `timeout` 설정이 필요하다.
