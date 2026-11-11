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

> `README.md`(구조·API·흐름)와 `README_DB.md`(DDL·상태 코드·S3 저장 위치)는 v2 스키마 기준으로 갱신돼 있다. 충돌 시 **코드가 기준**이다.

## 멀티 서비스 구성 내 위치

| 서비스 | 역할 | 포트 |
|--------|------|------|
| `aidea-api-workflow` | Argo 워크플로우 오케스트레이션 (본 서비스) | 5004 |
| `aidea-api-agent` | 에이전트 정의/실행 API + CLI (`cmd/workflow`가 Task Pod에서 본 서비스 콜백을 호출) | 5005 |
| `aidea-api-spinx` | LLM 게이트웨이 | - |
| `aidea-api-common` | 공용 라이브러리 | - |
| `hexim` | 공용 프레임워크 라이브러리 | - |

`go.work`가 `.`, `../aidea-api-common`, `../hexim`를 함께 묶는다. 짝궁 레포 `aidea-api-workflow-agent`가 Task Pod 안에서 도는 실행 바이너리(Data Plane)다.

## Control Plane ↔ Data Plane

- **본 서비스**: Workflow/Task DB 관리 + Argo DAG 빌드·제출 + run 상태 콜백 수신 (상시 구동 API 서버).
- **Task Pod**: Argo가 띄우는 컨테이너 안에서 1회성 실행. agent 실행 후 결과를 본 서비스 콜백 엔드포인트로 보고(stateless, DB 직접 접근 없음).
- 연결 고리: `service/argo/builder.go`가 Task 컨테이너에 `WORKFLOW_ID` / `RUN_ID` / `TASK_ID` / `WORKFLOW_STORAGE` / `WORK_DIR` / `MAX_RETRY` env를 주입한다. Pod는 이 env로 자신의 실행 컨텍스트를 식별한다.
- Pod 클라이언트(`aidea-api-agent/cmd/workflow`)의 계약 (유지해야 함):
  - 시작 시 `GET /workflow/{workflow_id}/run/{run_id}/task/{task_id}`로 자기 `task_id`의 run_task(task 정의 스냅샷 `agent_id`/`name`/`dependencies`/`task`)를 읽는다. run_task에는 별도 `id`가 없고 PK가 `(run_id, task_id)`다(`store.GetRunTask`도 이 PK로 조회). 행을 직접 만들지 않으며 실시간 `workflow_task`도 읽지 않는다.
  - `POST /-/run/task`로 상태를 **문자열**(`"RUNNING"`/`"SUCCEEDED"`/`"FAILED"`)로 갱신한다. 실패 시 `error_message`(실패 원인)를 함께 보내고, id를 얻기 전 실패해도 `run_id` + `task_id`로 보고한다.
  - 응답 필드 이름(`agent_id`, `name`, `dependencies`, `task`, `error_message`)을 바꾸면 Pod가 깨진다. API를 Pod 이미지보다 먼저 배포할 것.

## 기술 스택

- **웹 프레임워크**: Fiber v3 (`github.com/gofiber/fiber/v3`)
- **오케스트레이션**: Argo Workflows v4 (`github.com/argoproj/argo-workflows/v4`), Kubernetes `client-go`
- **DB/ORM**: MySQL + SQLBoiler v4 (`github.com/aarondl/sqlboiler/v4`), nullable는 `aarondl/null/v8`
- **S3**: `aws-sdk-go-v2` (`config`, `service/s3`) — debug 기록 업로드 (`service/storage`)
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
    index.go         # 라우트 트리 + 의존성 조립(store/argo/kube/debugStorage) + api 구조체 정의 + &api{...} 생성
    workflow.go      # Workflow/Task CRUD 핸들러 ((a *api) 메서드) + currentUserNo 헬퍼
    run.go           # Run/RunTask 핸들러 ((a *api) 메서드: createRun/getRun/finishRun/getRunTask/updateRunTask)
store/
  workflow.go           # SQLBoiler 기반 DB 접근 계층 (package store, type Workflow, 읽기/쓰기 DB 분리, 소유권 검증)
service/                # 외부 시스템 연동 서비스만 남김 (DB 접근 아님)
  argo/                 # builder.go(DAG 빌드) + index.go(Service: Argo client, RunWorkflow)
  kubernetes/           # index.go, pod.go, secret.go — client-go Pod/Secret 유틸 (+ pod_test.go)
  storage/              # debug.go — run 단위 debug 기록(S3) 업로드, best-effort
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
  - `DELETE /:workflow_id/task/:task_id` — soft delete, 다른 활성 task가 `dependencies`로 참조 중이면 409.
  - `/:workflow_id/run` POST(`createRun`) — Argo Workflow 생성/실행.
  - `/:workflow_id/run/:run_id/task` — run task 목록 조회, `/:task_id`로 단건 조회.
- `/-/run` — Argo 훅/Task Pod가 호출하는 내부 콜백. 호출 토큰의 사용자는 run 소유자여야 한다.
  - `POST /-/run/task` (`updateRunTask`) — run task 상태 **갱신** (insert 아님). `run_id` + `id` 또는 `task_id`로 미리 만들어진 행을 찾는다. 없으면 404.
  - `POST /-/run/:run_id/finish` (`finishRun`) — Argo exit 훅이 run 최종 상태 기록 + run_task 정리.
- 인증: `/`, `/health` 제외 전 라우트가 `middlewares.Authentication(jwt, redis, AccessRole...)` + `middlewares.RequestIdGenerator()` 통과. 사용자 번호는 `currentUserNo(c)` 헬퍼(`c.Value(types.ContextKeyUser).(*typ.User).No`).
- **소유권**: 모든 핸들러가 `userNo`를 store에 넘기고, store가 `user_no` 일치(+ 부모 workflow/run 소유)를 확인한다. 없거나 남의 리소스면 404.

## Run 실행 흐름 (`createRun`)

1. Workflow 조회(소유 확인) → 활성 Task 목록 조회(`GetTaskList`), 없으면 `er.New("no tasks to run")`.
2. `store.CreateRun`: `workflow_run`(PENDING, `instruction`)과 task별 `workflow_run_task`(PENDING, task 정의 스냅샷 복사)를 **한 트랜잭션**으로 insert.
3. 이후 단계 실패 시 `FinishRun(..., RunStatusError)`로 run을 ERROR 마감(남은 PENDING은 SKIPPED).
4. `UserToken`으로 사용자 토큰 조회 → `kubeService.MakeSecret(runID, token)`로 k8s Secret 생성.
5. `argoService.Build(wf, secret)`로 Argo `Workflow`(DAG) 구성 → `RunWorkflow`로 제출. 실패 시 Secret `DeleteSecret` 롤백.
6. Secret에 Argo Workflow를 소유자로 `PatchSecretOwnerReferences`(Workflow 삭제 시 GC).
7. run 상태를 `RUNNING` + `started_at`으로 갱신(`SetRun`). 이 단계 실패는 Workflow가 이미 실행 중이라 run을 마감하지 않고 에러만 반환(이후 exit hook이 최종 상태 기록).
8. 종료 시 Argo exit 훅이 `/-/run/{run_id}/finish`로 콜백 → `finishRun`이 `FinishRun` 호출.

### 종료 처리 (`finishRun` → `store.FinishRun`)

- Argo `workflow.status`를 `types.RunStatusFromArgo`로 변환 (`Succeeded`→30, `Failed`→40, `Error`→41, 그 외 99).
- run이 SUCCEEDED가 아니면 아직 `RUNNING`인 run_task를 `FAILED`로 (Pod가 보고 없이 죽은 경우), `PENDING`은 `SKIPPED`로. 한 트랜잭션.
- `failures`가 있으면 S3 `debug/failures.json`으로 업로드(best-effort, 실패/미설정이면 zap 경고 로그). **DB에는 저장하지 않는다.**
- SUCCEEDED면 user-token Secret 삭제.

### run_task 갱신 (`store.SaveRunTask`)

- 트랜잭션 + 행 잠금(`FOR UPDATE`)으로 PK `(run_id, task_id)` 행을 찾아(둘 중 하나라도 없으면 400) `status`, `started_at`, `finished_at`, `agent_run_id`, `error_message`만 반영(스냅샷·`retry_count`는 요청으로 변경 불가).
- **재시작 판정** (`isFinished`): 시작 신호(`PENDING` 또는 `RUNNING`)가 왔는데 기존 상태가 끝난 상태(SUCCEEDED/FAILED/ERROR/CANCELED/UNKNOWN)면 새 시도 → `retry_count + 1`, `error_message`/`started_at`/`finished_at` 초기화. PENDING/SKIPPED에서의 시작은 첫 실행, RUNNING→RUNNING은 중복 요청이라 세지 않는다(재시작은 run 종료 후에만 가능하고 종료 시 RUNNING은 FAILED로 정리되므로).
- `error_message`는 varchar(500)이라 `types.TruncateRunes`로 잘라 저장.

### Argo DAG 빌드 (`service/argo/builder.go`)

- Workflow 이름 `wf-{run_id}`, entrypoint `start`(DAG) + `exit`(HTTP 훅 템플릿).
- Task별 템플릿 `task-{task_id}`, `dependencies`는 `"task-"+id`로 접두어 붙여 DAG 의존성 구성. `dependencies`가 null/빈 값이면 선행 없는 task로 처리.
- 각 Task: `s3-sync` initContainer(root)가 `WORKFLOW_STORAGE`(`s3://{bucket}/{env}/workflow/{workflow_id}/run/{run_id}`)를 `--exclude "debug/*"`로 `WORK_DIR`(`/workspace`)에 sync + `output`·`debug` 디렉토리 생성 후 `chown 1000` → 본 컨테이너는 `task.Image`를 `RunAsUser 1000`으로 실행.
- EnvFrom: user-token Secret + `aidea-config-map` ConfigMap + `aidea-api-secret` Secret.
- Outputs(optional, 압축 없음): `output` = `/workspace/output` → `{s3Path}`, `debug` = `/workspace/debug` → `{s3Path}/debug`. 디렉토리 artifact는 키를 prefix로 쓰므로 **prefix는 `S3Bucket`이 아니라 `Key`에 붙인다**(`Key: s3Path + "/debug"`).
- `exit` 훅은 `http://{serviceName}.{namespace}.svc.cluster.local:5004/-/run/{run_id}/finish`로 POST(Authorization 헤더는 user-token Secret 참조), body는 `workflow.status`, `workflow.failures`.
- `maxRetry = "3"` 상수가 env `MAX_RETRY`로 전달됨(Pod는 읽지 않음).

## 코딩 관례

### 파일 네이밍 / DI
- `index.go` — 패키지 진입/배선. `Router(...)` 셋업 함수와 `router/workflow`의 `api` 구조체 정의를 담는다.
- `config/index.go`가 `ENV`(기본 `local`)로 YAML + 루트 `.env` 로드, `cfg.Get()` 싱글톤. `init()`은 `hexim.LoadEnv()` 호출.
- **DB 접근 계층은 `store` 패키지(`type Workflow`)**: `store.NewWorkflow(db, readDb *sql.DB)` — 쓰기(`anxer`)/읽기(`anxer_ro`) 커넥션 분리. aidea-api-agent의 `store.Agent`와 동일 패턴.
- `service/argo`, `service/kubernetes`, `service/storage`는 외부 시스템 client를 다루는 **진짜 서비스**라 `service/`에 둔다.
- **핸들러 DI는 구조체 방식**: `router/workflow`의 모든 핸들러(workflow.go + run.go)는 `api` 구조체의 메서드다 — `type api struct { workflowStore *store.Workflow; kubeService *kubernetes.Service; argoService *argo.Service; debugStorage *storage.Debug }`, 리시버 `(a *api)`. `Router()`가 조립해 `a := &api{...}`로 직접 생성하고 라우트를 `a.getRun` 식으로 등록한다. 구조체명은 `api`(소문자 비공개) — Fiber의 `Group(prefix, handlers ...any)` 인자와 겹치지 않도록. `debugStorage`는 없을 수 있으므로(`nil`) 사용 전 확인.

### 에러 처리
- `hexim/errors`를 별칭 `er`로 사용. 전파 `er.Wrap(err)`, 생성 `er.New("msg", er.WithCode(http.StatusBadRequest))`, 검사 `er.Is`, 필드 `er.WithKeyValue(...)`.
- 핸들러는 직접 에러 응답을 쓰지 않고 **에러를 반환** → `middleware.ErrorHandler()`가 중앙 렌더링.
- store는 `sql.ErrNoRows`를 `wrapNotFound`로 404 에러로 변환(`er.WithCode(http.StatusNotFound)`). 의존성 충돌은 409, 잘못된 상태 이름·누락된 식별자는 400.

### Fiber v3 핸들러
- 컨텍스트 값: `c.Value(types.ContextKeyXxx).(T)` 타입 단언. UUID 파라미터는 `middleware.BindUUIDParam`가 바인딩하므로 핸들러에서 `.(types.UUID)`. 옵셔널(목록 vs 단건)은 `c.Value(...)`가 `nil`인지로 분기.
- 바디 바인딩 `c.Bind().JSON(&x)`, 응답 `c.JSON(x)`, 빈 리스트는 `c.Status(http.StatusNoContent)`.
- 컨텍스트 키는 `types/context.go`에 문자열 상수로 집중 관리.

### 로깅
- zap 로거는 `config.Log.Logger()`로 획득(`aidea-api-common/logs`, info/error 로테이션). 핸들러 내 경고는 `zap.L()` 사용.

## 데이터베이스 / ORM

- **`v2_` 접두어 테이블** `v2_workflow`, `v2_workflow_task`, `v2_workflow_run`, `v2_workflow_run_task`를 쓴다. 생성 모델은 `database.V2Workflow` 등 `V2*`. 구 테이블은 폐기 예정이며 안정화 후 `RENAME TABLE`로 접두어를 제거한다. **rename 후 sqlboiler를 재생성하면 모델명이 `Workflow` 등으로 바뀌므로 `types/`, `store/`의 `database.V2*` 참조를 같이 고쳐야 한다.**
- 생성 코드는 `database/`에 있으며 **직접 수정 금지**, 스키마 변경 시 재생성. `sqlboiler.toml`은 DB 접속 정보가 있는 로컬 전용 파일이라 `.aiignore` 대상 → 열람·출력하지 않는다(셸 `cat`도 금지).
- **ID**: UUID v7(`types.NewUUID()`) → `BINARY(16)` 저장. 모델은 `[]byte`, 쿼리 시 `id[:]`, 복원은 `types.ToUUID([]byte)`. `types.UUID`는 `uuid.UUID` 별칭.
- **정렬**: 모든 목록은 `ORDER BY id DESC` (`created_at` 사용 안 함).
- **status**: DB는 `tinyint unsigned`(모델 `uint8`), 코드표는 `types/status.go`가 기준이며 코드 0은 "미지정".
  - 생명주기(workflow, workflow_task): `StatusDeleted=0`, `StatusActive=1`. 생성 시 요청값 무시, 항상 ACTIVE.
  - 실행(`types.RunStatus`): PENDING 10, RUNNING 20, SUCCEEDED 30, FAILED 40, ERROR 41, CANCELED 50, SKIPPED 60(run_task 전용), UNKNOWN 99. JSON에서는 문자열로 (역)직렬화, DB에는 정수. 새 상태는 `types/status.go`에서만 관리.
  - 업데이트 whitelist는 `Status != 0`으로 판단.
- **쿼리 스타일**: 타입드 query mod — `database.V2Workflows(database.V2WorkflowWhere.ID.EQ(id[:]), qm.OrderBy(...))`, 종단 `.One(ctx, exec)` / `.All(...)` / `.UpdateAll(ctx, exec, database.M{...})`.
- **읽기/쓰기 분리**: 읽기 `s.readDb`(anxer_ro), 쓰기 `s.db`(anxer). 쓰기 직후 복제 지연 주의 → 소유권 확인·갱신 전 조회는 `s.db`(또는 트랜잭션)로 한다.
- **upsert 패턴**: `ID == uuid.Nil`이면 `Insert(ctx, s.db, boil.Infer())` 후 생성 필드 역채움, 아니면 소유 확인 후 `Update(ctx, s.db, boil.Whitelist(...))`. whitelist는 `types.*.WhiteList()`가 변경된 필드만 수집(비면 "no value to change" 에러). 값이 NULL로 되돌려져야 하는 갱신(`SaveRunTask`의 재시작 초기화)은 whitelist 대신 행을 읽어 필드를 직접 조정한 뒤 컬럼 목록을 지정해 `Update`한다.
- **트랜잭션**: `CreateRun`(run + run_task), `Del`(workflow + task), `FinishRun`, `SaveRunTask`는 트랜잭션. 동시 갱신이 있는 곳은 `qm.For("UPDATE")`로 행 잠금.
- **soft delete**: `status`를 0으로 `UpdateAll`(workflow 삭제 시 하위 task도 함께).
- **도메인 ↔ 모델 변환**: `types/`의 `ToModel()` / `FromDatabaseObject()` / `ToWorkflowTask/Run/RunTask()` / `NewRunTask()`(task → 스냅샷). JSON 컬럼(dependencies/command)은 `null.JSON`, 도메인에서는 `json.RawMessage`로 다루고 `Commands()`가 언마샬.

## S3 저장 위치

`s3://{WORKFLOW_S3_BUCKET}/{env}/workflow/{workflow_id}/run/{run_id}/`

- `output/` 하위 — task 결과 파일(Argo output artifact, 후행 task가 sync로 받음)
- `debug/failures.json` — exit hook의 run 단위 failures, workflow API가 업로드(best-effort). API 서버는 S3 접근 가능.
- `debug/{task_id}_{n}.json`, `debug/{task_id}_{n}_failures.json` — 시도별 기록(`n` = `retry_count`), Pod가 `/workspace/debug`에 쓰면 Argo가 업로드
- `debug/`는 디버깅용이라 없어도 서비스가 동작해야 하고 Pod sync에서 제외된다. 서비스 데이터는 모두 DB에 둔다.

## 환경 설정 / 실행

필수 env (`.env` 또는 실행 환경):
- `ENV` — config 파일 선택 (`local`/`dev`/`live`, 기본 `local`)
- `AIDEA_ACCESS_ROLE` — 비면 panic
- `MYSQL_USER`, `MYSQL_PASSWORD`
- `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `VERTEX_CREDENTIALS` — LLM 프로바이더 (`VERTEX_CREDENTIALS`는 서비스 계정 JSON, project `anxer-ai`)
- `WORKFLOW_SERVICE_NAME`, `WORKFLOW_NAMESPACE`, `WORKFLOW_S3_BUCKET` — Argo 빌드/콜백 URL, S3 경로, debug 업로드(버킷 미설정이면 debug 업로드 비활성)
- `JWT_SECRET_KEY`, `APP_NAME`, `POD_IP`(헬스체크 응답용, 비면 "hello world")

```bash
cd /Users/phddy/Projects/dx/aidea-api-workflow
go mod download
go run main.go        # ENV=local 기본, config/local.yaml + .env
go build ./...        # 빌드 검증
go test ./...         # 테스트 (kubernetes pod_test 등)
```

- 로컬: `~/.kube/config` 있으면 사용, 없으면 in-cluster config. 로컬 모드에서는 prefork 비활성화.
- **Docker**: 멀티스테이지·멀티아치(amd64/arm64), `dumb-init` 엔트리포인트(Fiber prefork는 PID>1 필요).
- **CI/CD (Jenkinsfile)**: 병렬 아치 빌드 → Harbor push → 멀티아치 manifest → `argocd-apps` 리포 kustomize 이미지 태그 업데이트로 ArgoCD(GitOps) 배포.

## 알려진 개선 여지

- `task.ToModel`이 `Image`/`Command`를 요청값이 아닌 하드코딩(`amazon/aws-cli:2.28.12`, `["env"]`)으로 넣는다. 이미지 선택(python 버전 등)을 열 때는 요청에서 이미지를 직접 받지 말고 서버 허용 목록의 키로 변환할 것.
- run 재시작 시 run 자체 상태를 `RUNNING`으로 되돌리는 처리가 없다(run은 마지막 exit hook 결과로만 갱신).
- Task `dependencies` 검증이 없다(순환·존재하지 않는 ID는 Argo 제출 단계에서 실패하며 이때 run은 ERROR로 마감됨).
- Pod가 task 정의를 run_task 스냅샷에서 읽으므로 API가 스냅샷 필드를 내려주지 않으면 Pod가 `run task snapshot is empty`로 실패한다(배포 순서 주의).
- 소유권 검증은 `user_no` 일치만 확인하며 관리자 역할 예외가 없다.
- 보존 정책(run·`instruction`·S3 `debug/` lifecycle) 미정. `cmd/agent/`는 비어 있음.
