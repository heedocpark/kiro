---
inclusion: fileMatch
fileMatchPattern: 'aidea-api-workflow/**'
---

# AIDEA-API-WORKFLOW

AnXer/aidea 플랫폼의 **워크플로우 오케스트레이터(Control Plane)**. DB에 정의된 Workflow/Task를 받아 **Argo Workflows(DAG)** 오브젝트로 빌드·제출하고, 각 Task를 Kubernetes Pod(컨테이너)로 실행시킨다. 실행 상태(run/run_task)는 Task Pod와 Argo 라이프사이클 훅의 HTTP 콜백으로 다시 DB에 기록한다. 실제 LLM 에이전트 실행은 Task Pod 안에서 도는 짝궁 레포 `aidea-api-workflow-agent`(Data Plane)가 담당한다.

- **모듈 경로**: `github.com/mz-team-ax/aidea-api-workflow`
- **Go 버전**: 1.26.3
- **경로**: `/Users/phddy/Projects/dx/aidea-api-workflow`
- **HTTP 포트**: `0.0.0.0:5004`

> README.md는 상세하나 코드와 일부 어긋난다(ID는 HashId가 아니라 UUID v7 문자열, 내부 콜백은 `/-/run/...`, S3 env는 `WORKFLOW_S3_BUCKET`). 충돌 시 **코드가 기준**이다.

## 멀티 서비스 구성 내 위치

| 서비스 | 역할 | 포트 |
|--------|------|------|
| `aidea-api-workflow` | Argo 워크플로우 오케스트레이션 (본 서비스) | 5004 |
| `aidea-api-agent` | 에이전트 정의/실행 API + CLI | 5005 |
| `aidea-api-spinx` | LLM 게이트웨이 | - |
| `aidea-api-common` | 공용 라이브러리 | - |
| `hexim` | 공용 프레임워크 라이브러리 | - |

`go.work`가 `.`, `../aidea-api-common`, `../hexim`를 함께 묶는다. 짝궁 레포 `aidea-api-workflow-agent`가 Task Pod 안에서 도는 실행 바이너리(Data Plane)다.

## Control Plane ↔ Data Plane

- **본 서비스**: Workflow/Task DB 관리 + Argo DAG 빌드·제출 + run 상태 콜백 수신 (상시 구동 API 서버).
- **aidea-api-workflow-agent**: Argo가 띄우는 Task Pod 컨테이너 안에서 1회성 실행. SpinX 기반 LLM 에이전트 실행 + verifier 검증 후 결과를 본 서비스 콜백 엔드포인트로 보고(stateless).
- 연결 고리: `service/argo/builder.go`가 Task 컨테이너에 `WORKFLOW_ID` / `RUN_ID` / `TASK_ID` / `WORKFLOW_STORAGE` / `WORK_DIR` / `MAX_RETRY` env를 주입한다. agent는 이 env로 자신의 실행 컨텍스트를 식별한다.

## 기술 스택

- **웹 프레임워크**: Fiber v3 (`github.com/gofiber/fiber/v3`)
- **오케스트레이션**: Argo Workflows v4 (`github.com/argoproj/argo-workflows/v4`), Kubernetes `client-go`
- **DB/ORM**: MySQL + SQLBoiler v4 (`github.com/aarondl/sqlboiler/v4`), nullable는 `aarondl/null/v8`
- **LLM**: `hexim/llm` Store로 멀티 프로바이더(OpenAI/Anthropic/Bedrock/Google Vertex) 등록. 직접 SDK가 아닌 **SpinX 경유**가 원칙이나, `main.go`에서 프로바이더를 직접 등록한다(모델 사용은 에이전트 태스크용).
- **로깅**: Uber zap + 파일 로테이션 (`aidea-api-common/logs`)
- **설정/env**: `yaml.v3`, `joho/godotenv`, `aidea-api-common/configs`
- **UUID**: `google/uuid` (UUID v7)

## 디렉토리 구조

```
main.go              # 진입점: LLM 프로바이더(OpenAI/Anthropic/Bedrock/Google) 등록 + Fiber(:5004) 부트스트랩, graceful shutdown
config/
  index.go           # Config 로더 (ENV로 YAML 선택, AIDEA_ACCESS_ROLE 파싱 — 비면 panic)
  model.go           # 모델 상수 + alias 양방향 맵 (ModelName/ModelAlias)
  local|dev|live.yaml
router/
  index.go           # 최상위 Router(): /, /health, auth 미들웨어 + workflow 라우트 연결
  workflow/
    index.go         # 라우트 트리 + 서비스 init (workflowService/argoService/kubeService)
    workflow.go      # Workflow/Task CRUD 핸들러
    run.go           # Run/RunTask 핸들러 (createRun/getRun/finishRun/getRunTask/updateRunTask)
service/
  workflow/service.go   # SQLBoiler 기반 DB 접근 계층 (읽기/쓰기 DB 분리)
  argo/                 # builder.go(DAG 빌드) + index.go(Service: Argo client, RunWorkflow)
  kubernetes/           # index.go, pod.go, secret.go — client-go Pod/Secret 유틸 (+ pod_test.go)
database/            # SQLBoiler 생성 코드 — 직접 수정 금지
types/               # 도메인 타입 + 변환 헬퍼 (workflow/task/run/run_task/status/context/uuid)
middleware/          # exception.go(ErrorHandler), uuid.go(BindUUIDParam)
cmd/agent/           # 현재 비어 있음
Dockerfile           # 멀티스테이지 (golang-alpine → alpine), dumb-init 엔트리포인트
Jenkinsfile          # amd64/arm64 멀티아치 빌드 + ArgoCD(GitOps) 배포
```

## 라우트 (`router/workflow/index.go`)

사용자용 그룹 `/workflow`와 내부 콜백 그룹 `/-/run`이 분리돼 있다.

- `/workflow` — Workflow/Task/Run CRUD + `:id` 파라미터는 `middleware.BindUUIDParam(param, ContextKey)`로 `types.UUID`로 디코딩해 컨텍스트에 바인딩.
  - `/:workflow_id/run` POST(`createRun`) — Argo Workflow 생성/실행.
  - `/:workflow_id/run/:run_id/task` — run task 조회.
- `/-/run` — Argo 훅/Task Pod가 호출하는 내부 콜백.
  - `POST /-/run/task` (`updateRunTask`) — run task 상태 upsert.
  - `POST /-/run/:run_id/finish` (`finishRun`) — Argo exit 훅이 run 최종 상태 기록.
- 인증: `/`, `/health` 제외 전 라우트가 `middlewares.Authentication(jwt, redis, AccessRole...)` + `middlewares.RequestIdGenerator()` 통과. 사용자: `c.Value(types.ContextKeyUser).(*typ.User)`.

## Run 실행 흐름 (`createRun`)

1. Workflow 조회 → Task 목록 조회(`GetTaskList`), 없으면 `er.New("no tasks to run")`.
2. `types.Run{Status: WorkflowStatusPending}` 구성 → `SetRun`(insert, UUID v7 발급).
3. `UserToken`으로 사용자 토큰 조회 → `kubeService.MakeSecret(runID, token)`로 k8s Secret 생성.
4. `argoService.Build(wf, secret)`로 Argo `Workflow`(DAG) 구성 → `RunWorkflow`로 제출.
5. 실패 시 생성한 Secret을 `DeleteSecret`으로 롤백. 성공 시 Secret에 Argo Workflow를 소유자로 `PatchSecretOwnerReferences`(Workflow 삭제 시 GC).
6. run 상태를 `RUNNING`으로 갱신(`SetRun`).
7. 종료 시 Argo exit 훅이 `/-/run/{run_id}/finish`로 콜백 → `finishRun`이 상태/failures 반영, 성공이면 user-token Secret 삭제.

### Argo DAG 빌드 (`service/argo/builder.go`)

- Workflow 이름 `wf-{run_id}`, entrypoint `start`(DAG) + `exit`(HTTP 훅 템플릿).
- Task별 템플릿 `task-{task_id}`, `dependencies`는 `"task-"+id`로 접두어 붙여 DAG 의존성 구성.
- 각 Task: `s3-sync` initContainer(root)가 `WORKFLOW_STORAGE`(`s3://{bucket}/{env}/workflow/{workflow_id}/run/{run_id}`)를 `WORK_DIR`(`/workspace`)로 sync + `output` 디렉토리 생성 후 `chown 1000` → 본 컨테이너는 `task.Image`를 `RunAsUser 1000`으로 실행.
- EnvFrom: user-token Secret + `aidea-config-map` ConfigMap + `aidea-api-secret` Secret. Outputs: `/workspace/output`를 S3(ap-northeast-2)로 아카이브.
- `exit` 훅은 `http://{serviceName}.{namespace}.svc.cluster.local:5004/-/run/{run_id}/finish`로 POST(Authorization 헤더는 user-token Secret 참조).
- 주의: `maxRetry = "3"` 상수가 env `MAX_RETRY`로 전달됨. (README의 "maxRetry 미설정" 설명은 현행 코드와 다름 — 코드 기준.)

## 코딩 관례

### 파일 네이밍 / DI
- `index.go` — 패키지 진입/배선. 패키지 `init()`, 공유 var, `Router(...)`/`NewService(...)`를 담는다.
- **패키지 레벨 싱글톤 + 패키지별 `init()` 배선**. `router/workflow/index.go`의 `init()`이 `workflowService`/`argoService`/`kubeService`를 생성, 실패 시 `log.Panic`/`log.Fatal`.
- `config/index.go`가 `ENV`(기본 `local`)로 YAML + 루트 `.env` 로드, `cfg.Get()` 싱글톤. `init()`은 `hexim.LoadEnv()` 호출.
- 서비스 생성자: `workflow.NewService(apiConfig, db, readDb *sql.DB)` — 쓰기(`anxer`)/읽기(`anxer_ro`) 커넥션 분리.

### 에러 처리
- `hexim/errors`를 별칭 `er`로 사용. 전파 `er.Wrap(err)`, 생성 `er.New("msg", er.WithCode(http.StatusBadRequest))`, 검사 `er.Is`, 필드 `er.WithKeyValue(...)`.
- 핸들러는 직접 에러 응답을 쓰지 않고 **에러를 반환** → `middleware.ErrorHandler()`가 중앙 렌더링.
- `sql.ErrNoRows`는 서비스에서 `er.New("not found ...", er.WithCode(http.StatusNotFound))`로 변환.

### Fiber v3 핸들러
- 컨텍스트 값: `c.Value(types.ContextKeyXxx).(T)` 타입 단언. UUID 파라미터는 `middleware.BindUUIDParam`가 바인딩하므로 핸들러에서 `.(types.UUID)`. 옵셔널(목록 vs 단건)은 `c.Value(...)`가 `nil`인지로 분기.
- 바디 바인딩 `c.Bind().JSON(&x)`, 응답 `c.JSON(x)`, 빈 리스트는 `c.Status(http.StatusNoContent)`.
- 컨텍스트 키는 `types/context.go`에 문자열 상수로 집중 관리.

### 로깅
- zap 로거는 `config.Log.Logger()`로 획득(`aidea-api-common/logs`, info/error 로테이션).

## 데이터베이스 / ORM

- **SQLBoiler v4**, `sqlboiler.toml` whitelist: `workflow`, `workflow_task`, `workflow_run`, `workflow_run_task`. 생성 코드는 `database/`에 있으며 **직접 수정 금지**, 스키마 변경 시 재생성.
- **ID**: UUID v7(`types.NewUUID()`) → `BINARY(16)` 저장. 모델은 `[]byte`, 쿼리 시 `id[:]`, 복원은 `types.ToUUID([]byte)`. `types.UUID`는 `uuid.UUID` 별칭.
- **쿼리 스타일**: 타입드 query mod — `database.Workflows(database.WorkflowWhere.ID.EQ(id[:]), qm.OrderBy(...))`, 종단 `.One(ctx, s.readDb)` / `.All(...)` / `.UpdateAll(ctx, s.db, database.M{...})`.
- **읽기/쓰기 분리**: 읽기 `s.readDb`(anxer_ro), 쓰기 `s.db`(anxer). 쓰기 직후 복제 지연 주의.
- **upsert 패턴**: `ID == uuid.Nil`이면 `Insert(ctx, s.db, boil.Infer())` 후 생성 필드 역채움, 아니면 `Update(ctx, s.db, boil.Whitelist(...))`. whitelist는 `types.*.WhiteList()`가 변경된 필드만 수집(비면 "no value to change" 에러).
- **soft delete**: `status` 컬럼을 0으로 `UpdateAll`(워크플로우 삭제 시 하위 task도 함께, 트랜잭션).
- **도메인 ↔ 모델 변환**: `types/`의 `ToModel()` / `FromDatabaseObject()` / `ToWorkflowTask/Run/RunTask()`. JSON 컬럼(dependencies/command)은 `null.JSON`, 도메인에서는 `json.RawMessage`/`[]byte`로 다루고 `Dependencies()`/`Commands()`가 언마샬.

## 환경 설정 / 실행

필수 env (`.env` 또는 실행 환경):
- `ENV` — config 파일 선택 (`local`/`dev`/`live`, 기본 `local`)
- `AIDEA_ACCESS_ROLE` — 비면 panic
- `MYSQL_USER`, `MYSQL_PASSWORD`
- `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `VERTEX_CREDENTIALS` — LLM 프로바이더 (`VERTEX_CREDENTIALS`는 서비스 계정 JSON, project `anxer-ai`)
- `WORKFLOW_SERVICE_NAME`, `WORKFLOW_NAMESPACE`, `WORKFLOW_S3_BUCKET` — Argo 빌드/콜백 URL 및 S3 경로
- `JWT_SECRET_KEY`, `APP_NAME`, `POD_IP`(헬스체크 응답용, 비면 "hello world")

```bash
cd /Users/phddy/Projects/dx/aidea-api-workflow
go mod download
go run main.go        # ENV=local 기본, config/local.yaml + .env
go build ./...        # 빌드 검증
go test ./...         # 테스트 (kubernetes pod_test 등)
```

> 현재 `go build ./...`는 `config.Log.Logger undefined`로 실패할 수 있다(`aidea-api-common`의 `logs.LogConfig` 버전과 `main.go` 호출부 불일치). `go.work`가 묶는 로컬 `aidea-api-common` 버전을 맞추거나 `main.go`의 로거 획득부를 공통 라이브러리 시그니처에 맞춰야 한다.

- 로컬: `~/.kube/config` 있으면 사용, 없으면 in-cluster config. 로컬 모드에서는 prefork 비활성화.
- **Docker**: 멀티스테이지·멀티아치(amd64/arm64), `dumb-init` 엔트리포인트(Fiber prefork는 PID>1 필요).
- **CI/CD (Jenkinsfile)**: 병렬 아치 빌드 → Harbor push → 멀티아치 manifest → `argocd-apps` 리포 kustomize 이미지 태그 업데이트로 ArgoCD(GitOps) 배포.

## 알려진 개선 여지

- `router/workflow/workflow.go`의 `deleteTask` 핸들러가 미구현(`return nil`) 가능성 — 수정 전 확인.
- `cmd/agent/` 디렉토리가 비어 있음.
- 서비스 쿼리의 상당수가 `userNo` 소유자 필터를 주석 처리(`//database.WorkflowWhere.UserNo.EQ(...)`)해 둔 상태 — 소유권 검증 공백.
- `task.ToModel`이 `Image`/`Command`를 요청값이 아닌 하드코딩(`amazon/aws-cli:2.28.12`, `["env"]`)으로 넣는다 — 실제 태스크 이미지/커맨드 반영 전 확인.
