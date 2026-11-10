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
  agent/         # 핸들러 ((a *api) 메서드): agent.go, run.go, version.go, subscribe.go
                 #   index.go (api 구조체 정의 + 라우트 트리 + &api{...} 조립), type.go (기본 프롬프트 템플릿/상수)
store/           # agent.go — DB 접근 계층 (package store, type Agent, 모든 쿼리 SQLBoiler)
agent/           # 실행 엔진: chat.go, runner.go, worker.go, verifier.go,
                 #   orchestrator.go, prompt.go, parse.go, history.go,
                 #   response_format.go, spinx.go, type.go, index.go (+ *_test.go)
types/           # 도메인/요청/응답 타입 + 변환 헬퍼 (uuid.go, context.go 등)
database/        # SQLBoiler 생성 코드 — 직접 수정 금지
middleware/      # exception.go — Fiber ErrorHandler()
config/          # index.go (Config 로더) + local.yaml, dev.yaml, live.yaml
```

## 실행 엔진 (`agent/` 패키지)

각 구성요소(Worker/Verifier/Runner/Orchestrator/Coordinator/Chat/Recorder)의 인터페이스·설정·동작 상세는 **`aidea-api-agent-engine.md`** 참고(`agent/` 하위 파일을 열면 자동 로드). 2계층 오케스트레이션 조립(stage/fanout)과 토큰·품질 디버깅 교훈은 **`aidea-api-agent-orchestration.md`** 참고.

## 코딩 관례

### 파일 네이밍
- `index.go` — 패키지 진입/배선 파일. `Router(...)` 셋업 함수와 `router/agent`의 `api` 구조체 정의를 담는다.
- `type.go` — 패키지 로컬 상수와 작은 타입.
- 그 외 파일은 도메인 관심사별 명명 (`agent.go`, `run.go`, `worker.go` 등).

### DI 스타일
- DB 접근 계층은 `store` 패키지(`type Agent`)로 분리하고 명시적 생성자 `store.NewAgent(db, readDb *sql.DB)` 사용 — 쓰기(`anxer`)/읽기(`anxer_ro`) 커넥션 분리. (이전 `service/agent`의 `NewService`/`Service`에서 이름을 바꿈 — stutter 회피 + 실행 엔진 `agent` 패키지와의 혼동 제거.)
- **핸들러 DI는 구조체 방식**: `router/agent`의 핸들러는 `api` 구조체 메서드다 — `type api struct { agentStore *store.Agent }`, 리시버 `(a *api)`. `Router()`가 `store.NewAgent(...)`로 store를 만들어 `a := &api{agentStore: ...}`로 직접 생성(재사용 없어 `NewAPI` 생성자 없음)하고, 라우트를 `a.get`/`a.invoke` 식으로 등록한다. 구조체명은 `api`(소문자 비공개) — Fiber의 `Group(prefix, handlers ...any)` 미들웨어 인자 `handlers`와 겹치지 않도록. 공통 규칙은 `go-conventions.md`의 "HTTP 핸들러 DI" 참고.
  - 예외: `subscribe(add bool)`는 add/del 두 라우트를 한 핸들러로 처리하려고 `(a *api) subscribe(add bool) func(fiber.Ctx) error` 꼴(메서드 + 인자 받는 클로저 반환). `remove`는 no-op 스텁이라 아직 `api` 메서드가 아닌 패키지 함수로 남아 있다(store 미사용).
  - `config/index.go`는 `ENV`로 선택된 YAML + `.env`를 로드해 `cfg.Get()` 싱글톤으로 노출(`init()`에서 `hexim.LoadEnv()`). store/엔진 조립이 아직 `Router()` 안에 있어 완전한 composition root(`main` 주입)로는 추가 전환 여지가 있다.

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

`router/index.go` → `router/agent/index.go` 라우트 트리 → `router/agent/*.go` 핸들러 → `store`(DB) 및/또는 `agent/`(실행 엔진) → `types.ToXxx` 변환 → `c.JSON`. 에러는 `er.Wrap`/`er.New`로 버블업되어 `middleware.ErrorHandler()`가 렌더링. (핸들러의 store 핸들은 `agentStore` 변수.)

- **인증**: `/`, `/health` 제외 모든 라우트가 `middlewares.Authentication(...)`(JWT + Redis 세션, `AIDEA_ACCESS_ROLE`로 역할 필터) + `middlewares.RequestIdGenerator()` 통과. 사용자: `c.Value(types.ContextKeyUser).(*typ.User)`.
- **프롬프트 조합**: 엔진 기본 프롬프트가 우선. 버전의 `system_prompt`를 `{{AGENT_SYSTEM_PROMPT}}` placeholder에 주입(`systemPrompt.Make()`). 충돌하는 사용자 지시는 무시.
- **Run 저장**: 앱 생성 UUID로 `types.AgentRun` 구성 → 엔진 실행 → `agentStore.SetRun(...)`. `agent_run`은 `s.db`로 먼저 insert하고, `usages`가 있으면 별도 트랜잭션으로 모델별 `agent_usage`를 넣는다(usage는 보조 데이터라 run과 원자적으로 묶지 않음 — 의도된 설계). 저장 실패해도 응답은 반환하되 `zap.L().Error`로 로깅한다. `history`는 DB에만 저장, 응답에서는 제거(`run.History = nil`).
- **SSE 스트리밍**: `text/event-stream` 헤더, `c.SendStreamWriter`, `data: {json}\n\n` 프레임, 5분 타임아웃, panic recover. 첫 이벤트 `id`를 run UUID로 교체. 스트림 모드는 검증 없음(`Stream.Enable && Verifier` 조합은 400). `NewChat`은 `version.Model`을 쓴다(과거 `VerifierModel` 오용 버그는 수정됨).

## 워크플로우 Task 간 데이터 전달

워크플로우에서 에이전트는 task로 쓰이고, 선행 task 결과가 후행 task의 `upstream`으로 주입된다. `cmd/workflow/service.go:Run`이 `Load()`로 의존 task 결과 파일(`run-{runID}-task-{depID}.json`)을 읽어 `[]agent.Upstream`을 만들고 `Input{RunInput, Task, Goal, Upstreams}`로 Coordinator에 넘긴다. 세 진입점(API invoke / agent CLI / workflow CLI)의 조립 차이와, workflow CLI에 적용된 2계층 구조는 `aidea-api-agent-orchestration.md` 참고.

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

## 알려진 개선 여지 / 버그

> `cmd/workflow` 쪽 항목은 2계층 재설계로 상당 부분 해소됨 — 최신 상태는 `aidea-api-agent-orchestration.md`의 "남은 과제" 참고. 아래는 그 외 전반(주로 API invoke/agent CLI, DB, CI) 항목.

1. `DELETE /agent/{id}`의 `remove()`, `deleteVersion()`이 nil 반환 no-op 스텁(미구현).
2. **workflow/agent CLI 에러에 token이 로깅될 수 있음** — `er.WithKeyValue("body", res.Body())` 등으로 응답 본문이 필드에 실림. go-conventions의 "토큰·시크릿 로그 금지"와 충돌. (검토 중)
3. `version_no` 발급 시 `SELECT ... FOR UPDATE` 행 잠금 부재(동시성 gap, README_DB §4-2).
4. `sqlboiler.toml`에 운영 DB 자격증명이 평문 — 환경변수 분리 권장.
5. **invoke는 단일 Runner 전용으로 정리됨** — 오케스트레이션(2계층)은 즉석 HTTP 요청 성격에 안 맞고(비용·단계 의존 없음) 호출자가 전략을 판단할 수 없어, `request.Orchestrator` 분기를 제거했다. invoke는 Worker(+조건부 Verifier) 단일 Runner만 돈다. 2계층은 `cmd/workflow` 전용. agent CLI(`cmd/agent`)도 단일 Runner. 조립 전략을 열려면 요청 플래그가 아니라 **에이전트 버전 설정**으로 받는 방향이 맞다(남은 과제).

### 해소됨 (이력)
- 스트림 모드가 `model` 대신 `verifier_model`을 쓰던 버그 → `NewChat(Model: version.Model)`로 수정.
- invoke의 Coordinator가 `Parallel` 미지정으로 순차 실행되던 회귀 → Coordinator 분기 자체를 제거하며 소멸.
- DB 접근 계층 `service/agent`(`NewService`/`Service`) → `store`(`store.NewAgent`/`store.Agent`)로 리네임.
