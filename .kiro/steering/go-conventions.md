---
inclusion: fileMatch
fileMatchPattern: '**/*.go'
---

# GO 개발 컨벤션 (공통)

`aidea-api-*` 서비스군과 공용 라이브러리(`aidea-api-common`, `hexim`)가 공유하는 Go 컨벤션. 특정 서비스의 상세 구조/흐름은 각 프로젝트 steering 파일을 따르고, 여기서는 **모든 Go 코드에 공통으로 적용되는 규칙**만 다룬다.

> 컨벤션과 기존 코드가 충돌하면 **그 레포에서 다수가 쓰는 실제 코드가 기준**이다. 새 코드를 쓸 때는 주변 파일의 관례를 먼저 보고 맞춘다.

## 기본 환경

- **Go 버전**: 1.26.3 (Alpine 3.23 기반 빌드)
- **모듈 경로**: `github.com/mz-team-ax/<repo>`
- **멀티 모듈**: `go.work`로 서비스 + `aidea-api-common` + `hexim`을 로컬에서 함께 묶는다. 공용 라이브러리 수정 시 소비 레포도 함께 빌드 검증한다.
- 포맷팅은 `gofmt`/`goimports` 기준(탭 들여쓰기). 커밋 전 `go build ./...`로 검증한다.

## 패키지 / 파일 네이밍

- `index.go` — 패키지 진입/배선 파일. `Router(...)`/`NewService(...)` 같은 셋업 함수를 담는다. **새 패키지를 만들면 이 관례를 따른다.** 단, 배선은 패키지 `init()`/전역 var가 아니라 **인자로 의존성을 받는 함수**로 노출한다(아래 DI 참고).
- `type.go` — 패키지 로컬 상수와 작은 타입.
- 그 외 파일은 도메인 관심사별로 명명한다 (`agent.go`, `run.go`, `worker.go`, `builder.go` 등). "한 파일에 한 책임".
- 테스트는 대상 코드 옆에 `*_test.go`로 둔다.

## 의존성 주입 (DI)

**명시적 생성자 + `main`에서 주입**이 원칙. 의존성은 생성자 인자로 받고, `main`을 조립 지점(composition root)으로 삼아 위에서 아래로 넘긴다. `net/http`/`database/sql`이 값을 반환해 호출자가 넘기는 것과 같은 철학이고, 규모가 커지면 생성자 기반을 유지한 채 `wire`(컴파일 타임)나 `fx`(런타임) 같은 DI 도구로 조립을 자동화한다.

- **`init()` 배선과 패키지 레벨 싱글톤은 지양한다.** 테스트에서 교체가 어렵고, 초기화 순서가 불명확하며, 숨은 결합을 만든다.
- 서비스 생성자는 명시적 시그니처를 쓴다: `NewService(db, readDb *sql.DB) (*Service, error)`.
- 라우터도 `init()`이 아니라 필요한 의존성을 **인자로 받는 함수**로 만든다: `Router(xxxService *xxx.Service) ...`. `main`이 서비스를 생성해 라우터에 넘긴다.
- 조립 중 실패(DB 연결, 필수 설정 누락 등)는 `main`에서 에러를 받아 `log.Fatal`로 부팅을 중단한다. 패키지 `init()`에서 `log.Panic`하지 않는다.

```go
// main.go (composition root)
cfg := config.Load()              // ENV 기반 YAML + .env 로드
db, readDb := database.Open(cfg)  // 쓰기/읽기 커넥션

agentService, err := agent.NewService(db, readDb)
if err != nil {
    log.Fatal(err)
}

app := router.Router(agentService) // 의존성을 인자로 주입
log.Fatal(app.Listen(cfg.Addr))
```

> **레거시 참고**: 기존 `router/<domain>/index.go`는 `init()`에서 서비스 싱글톤을 만들고, `config/index.go`는 `cfg.Get()` 싱글톤 + `init()`에서 `hexim.LoadEnv()`를 호출한다. 이 패턴은 **레거시로만 인정**하며 신규 코드에서는 쓰지 않는다. 기존 코드를 수정할 때 여력이 되면 명시적 주입으로 점진 전환한다.

## 에러 처리

`hexim/errors`를 import 별칭 **`er`**로 통일한다.

```go
import er "github.com/mz-team-ax/hexim/errors"

return er.Wrap(err)                                       // 전파
return er.New("msg", er.WithCode(http.StatusBadRequest))  // 생성 (+ HTTP 코드)
er.Is(err, target) / er.As(err, &target)                  // 검사
er.New("...", er.WithKeyValue("key", val))                // 필드 부착
```

- **모든 에러는 경계를 넘을 때 `er.Wrap(err)`로 감싼다** (서비스 → 핸들러). 원본 에러를 그대로 반환하지 않는다.
- 핸들러는 직접 에러 응답을 쓰지 않고 **에러를 반환**한다. `middleware.ErrorHandler()`가 중앙에서 `er.Error`를 풀어 `{id, message, detail}` 응답으로 렌더링한다 (404/401/403은 로깅 생략).
- `sql.ErrNoRows`는 서비스 계층에서 `er.New("not found ...", er.WithCode(http.StatusNotFound))`로 변환한다.

## 로깅

- **Uber zap** (`go.uber.org/zap`) + 파일 로테이션(`aidea-api-common/logs`, iso8601). 전역 로거는 `zap.L()`로 획득한다.
- `fmt.Println`/`println`/`log.Printf`를 영구 코드에 남기지 않는다. (panic recover 로깅 등 불가피한 경우는 예외.)
- 토큰·시크릿·user-token을 로그에 남기지 않는다.

## 설정 / 환경

- `ENV`(`local`/`dev`/`live`, 기본 `local`)로 `config/<env>.yaml`을 선택하고 루트 `.env`를 자동 로드한다.
- 필수 env가 비면 **부팅 시 panic**시킨다 (예: `AIDEA_ACCESS_ROLE`). 런타임에 조용히 넘어가지 않는다.
- 설정은 `yaml.v3` + `joho/godotenv` + `aidea-api-common/configs`.

## 웹 계층 (Fiber v3)

프레임워크는 **Fiber v3** (`github.com/gofiber/fiber/v3`).

- 핸들러 시그니처는 `func(c fiber.Ctx) error`. 컨텍스트는 `ctx := c.Context()`.
- 컨텍스트 값: `c.Value(types.ContextKeyXxx).(T)` 타입 단언. 컨텍스트 키는 `types/context.go`에 문자열 상수로 집중 관리한다.
- 옵셔널 파라미터(목록 vs 단건)는 `c.Value(...)`가 `nil`인지로 분기한다.
- 바디 바인딩 `c.Bind().JSON(&x)`, 응답 `c.JSON(x)`, 빈 리스트는 `c.Status(http.StatusNoContent)`(204).
- 인증: `/`, `/health`를 제외한 라우트는 `middlewares.Authentication(...)`(JWT + Redis 세션, `AIDEA_ACCESS_ROLE` 역할 필터) + `middlewares.RequestIdGenerator()`를 통과한다. 사용자: `c.Value(types.ContextKeyUser).(*typ.User)`.
- 핸들러 지역 변수는 상단 `var (...)` 블록으로 모아 선언하는 스타일을 따른다.

## 데이터베이스 / ORM (SQLBoiler v4)

- **SQLBoiler v4** (`github.com/aarondl/sqlboiler/v4`) + MySQL, nullable는 `aarondl/null/v8`.
- `database/`의 생성 코드는 **직접 수정 금지**. 스키마 변경 시 `sqlboiler.toml` whitelist 기준으로 재생성한다.
- **읽기/쓰기 분리**: 읽기는 `s.readDb`(`anxer_ro`), 쓰기는 `s.db`(`anxer`). 쓰기 직후 복제 지연에 주의한다.
- **쿼리 스타일**: 타입드 query mod 사용.
  ```go
  database.Agents(
      database.AgentWhere.ID.EQ(id[:]),
      qm.Select(database.AgentColumns.Name),
      qm.OrderBy(database.AgentColumns.CreatedAt + " DESC"),
  ).One(ctx, s.readDb) // .All / .Count / .UpdateAll(ctx, s.db, database.M{...})
  ```
- **upsert 패턴**: `ID == uuid.Nil`이면 `Insert(ctx, s.db, boil.Infer())` 후 생성 필드 역채움, 아니면 `Update(ctx, s.db, boil.Whitelist(...)/x.WhiteList())`. whitelist는 변경된 필드만 수집(비면 "no value to change").
- **soft delete**: `status` 컬럼 값을 바꿔 처리하고, 하위 레코드가 있으면 트랜잭션으로 함께 갱신한다.

## ID / UUID

- **ID는 UUID v7** (`google/uuid`). 앱에서 생성(`types.NewUUID()`), DB에는 `BINARY(16)` 저장.
- SQLBoiler 모델 필드는 `[]byte`. 쿼리 시 `id[:]`, 복원은 `types.ToUUID([]byte)`. nullable UUID는 `null.Bytes`.
- `types.UUID`는 `uuid.UUID`의 별칭.

## 도메인 ↔ 모델 변환

- DB 모델과 도메인 타입을 분리하고 `types/`의 변환 헬퍼로 오간다: `ToModel()` / `FromDatabaseObject()` / `ToXxx()`.
- JSON 컬럼은 모델에서 `null.JSON`, 도메인에서는 `json.RawMessage`/`[]byte`로 다루고 전용 언마샬 메서드로 푼다.

## 테스트

- 표준 `testing` + `stretchr/testify`. 테스트는 대상 패키지 안, 코드 옆에 둔다.
- `*_Live` 네이밍 테스트는 실제 외부 서비스(SpinX/MCP 등)를 호출한다 → **의도적으로만 실행**, 일반 CI에서 돌리지 않는다.
- 기능 추가·버그 수정 시 관련 테스트를 함께 작성/갱신하고, `go test ./...`로 검증한다.

## 빌드 / 실행

```bash
go mod download
go run .          # 또는 go run main.go (ENV=local 기본)
go build ./...    # 빌드 검증 (커밋 전 필수)
go test ./...     # _Live 테스트는 외부 호출 → 의도적으로만
```

- **Docker**: 멀티스테이지·멀티아치(amd64/arm64), `dumb-init` 엔트리포인트(Fiber prefork는 PID>1 필요). 베이스 이미지는 Harbor(`ax-harbor.cjmezzomedia.com`).
- **CI/CD (Jenkinsfile)**: 병렬 아치 빌드 → Harbor push → 멀티아치 manifest → `argocd-apps` 리포 kustomize 이미지 태그 업데이트로 ArgoCD(GitOps) 배포.

## LLM 호출 원칙

- 서비스 코드에서 LLM SDK를 직접 쓰지 않고 **모든 LLM 호출은 SpinX 게이트웨이를 경유**하는 것이 원칙이다 (`hexim/llm` Store 등록). 직접 프로바이더를 등록하는 예외(워크플로우 태스크용 등)는 해당 레포 steering을 따른다.
