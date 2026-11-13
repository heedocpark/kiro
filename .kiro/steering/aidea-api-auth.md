---
inclusion: fileMatch
fileMatchPattern: 'aidea-api-auth/**'
---

# AIDEA-API-AUTH

CJ MezzoMedia AI 챗봇 **앤써(AnXer)**의 인증 서비스. 사내 인사 DB(datamax)로 로그인을 검증하고, JWT 발급 + Redis 세션으로 토큰/사용자 세션을 관리한다. 다른 `aidea-api-*` 서비스는 `aidea-api-common/middlewares.Authentication`으로 **같은 JWT 키·같은 Redis 세션 키**를 공유해 이 서비스가 발급한 토큰을 검증한다.

- **모듈 경로**: `github.com/mz-team-ax/aidea-api-auth`
- **Go 버전**: 1.26.3 (Alpine 3.23)
- **경로**: `/Users/phddy/Projects/dx/aidea-api-auth`
- **HTTP 포트**: `:5001` (`main.go`에 하드코딩), `APP_NAME=anxer-api-auth`
- **API 명세**: 레포 `openapi.yaml`이 1차 소스. `ENV != live`일 때 `/auth/docs`(Swagger UI), `/auth/openapi.yaml`로 노출. 단 `/api/user/token`은 openapi에 **없다**(k8s 내부통신용).

> 공통 Go 컨벤션은 `go-conventions.md`를 따른다. 이 레포는 그 컨벤션의 **레거시 패턴**(init/전역 var 싱글톤, 패키지 var 배선)을 그대로 쓰는 가장 오래된 축이다. 새 코드는 컨벤션을, 기존 코드 수정은 주변 관례를 따른다(아래 "컨벤션과의 차이" 참고).

## 레이어 구조

`store`/`service` 계층이 **없다.** 핸들러가 sqlboiler/Redis를 직접 호출한다.

- **`main.go`** — Fiber 앱 생성, CORS/RequestId 미들웨어, `errors.Handler()` 등록, graceful shutdown(`ENV != local`), prefork(`PROCESS > 1`).
- **`router/index.go`** — `Router(app)`: `/health`(POD_IP 응답) + 하위 라우터 조립.
- **`router/auth`** — 로그인/로그아웃/갱신/검증/관리자 토큰 발급/로그인 이력.
  - `index.go` 핸들러 + `Router`, `login.go` 로그인 파이프라인 함수들, `history.go` 응답 변환, `model.go` `JwtMake` 요청 바디.
- **`router/user`** — 사내 사용자 조회/검색 (datamax 읽기 전용).
- **`router/api`** — `RoleApi` 토큰으로 **사용자 토큰 발급**(서비스 간 내부통신). 핸들러가 `Router()` 안에 인라인 클로저.
- **`database/anxer`** — SQLBoiler 생성: `user_login_log`만 (whitelist).
- **`database/datamax`** — SQLBoiler 생성 + 수기 변환: `dmp_user_inner`(메조), `dxe_user_inner`(DXE) → `FromDatabaseMezzoUser`/`FromDatabaseDxeUser`로 `types.User` 변환.
- **`errors/`** — 인증 전용 에러 상수 + 중앙 `Handler()`.
- **`config/`** — `ENV`별 YAML(`local`/`dev`/`live`).

## 엔드포인트

| 경로 | 인증 | 설명 |
|------|------|------|
| `GET /health` | 없음 | POD_IP 문자열 |
| `POST /auth/login` | 없음 | 로그인 → `types.User` + JWT |
| `POST /auth/refresh` | Bearer | 새 토큰 발급 + 기존 세션 삭제 |
| `GET /auth/validate` | Bearer | 토큰 검증, 현재 사용자 반환 |
| `POST /auth/logout` | Bearer | 현재 토큰 세션 삭제 |
| `GET /auth/user/history` | Bearer | 최근 90일 로그인 이력 |
| `POST /auth/make` | Bearer + `ADMIN` | 역할 지정 토큰 발급 (`userNo=-1`) |
| `GET /user/:company/:no` | Bearer(`AccessRole`+`API`) | 사용자 단건 |
| `GET /user/:company/search` | Bearer(`AccessRole`+`API`) | `user_name`/`team_name` like 검색 (둘 중 하나 필수) |
| `POST /api/user/token` | Bearer(`API` 역할) | 사용자 no로 JWT 발급 (k8s 내부) |

- `company`는 `m`(메조, `dmp_user_inner`) / `d`(DXE, `dxe_user_inner`). 대소문자 무시.
- `/api/user/token`은 `company`를 요청에서 받지 않고 **`user.No > 20000`이면 `d`, 아니면 `m`**으로 추정한다(번호 대역 규칙). 번호 체계가 바뀌면 여기를 같이 고친다.

## 로그인 흐름

`login`은 `loginFn` 파이프라인을 순서대로 실행하고 **첫 에러에서 중단**한다.

```
failCheck → userLogin → clearFailCount → userPut → makeToken → lastLoginHistory
```

- **`failCheck`** — Redis `user_fail` 카운터가 **2 초과(= 3회 실패)면** `TooManyFailedLogin`.
- **`userLogin`** — datamax에서 `status='1'` 사용자 조회(`id` 또는 `no`) → 비밀번호 검증. **`cfg.IsLocal`이면 비밀번호 검증을 건너뛴다.** DXE(`d`)는 `mandatory_consent_yn='1'`(필수 동의)이 아니면 `UserIdNotFound`.
- **`clearFailCount`** — 성공 시 실패 카운터 삭제.
- **`userPut`** — `user_session` 키(`{company}:{no}`)에 `types.User`를 Redis 저장(TTL = `jwt.exp`).
- **`makeToken`** — 역할 계산(`cfg.Role.MakeRole(company, id)`) → JWT 발급 → `token_session` 키(`sessionId`)에 `{no, company, role}` 저장(TTL = `jwt.exp`).
- **`lastLoginHistory`** — 직전 성공 로그인의 IP/시각을 `user`에 채운다(`anxer_ro`).

로그인 후 처리:
- 실패가 `IncorrectPassword`일 때만 `failCount`로 카운터 증가. (사용자 없음/조회 실패는 카운트하지 않는다.)
- **로그인 로그는 고루틴으로 비동기 기록**: `user_login_log` insert + 파일 로그(`auth.log`, 구분자 `L`). `user.No > 0 && ip != 127.0.0.1`일 때만. 응답을 막지 않는다.
- `refresh`는 `userPut → makeToken → deleteCurrentToken`(새 토큰 발급 후 이전 세션 삭제), 파일 로그 구분자 `R`.

### 세션 / 토큰 구조

- **JWT 클레임**: `id`(= sessionId, uuid), `no`, `company`, `role[]`, `exp`, `iat`. HS256, 키는 `jwt.key`(`${JWT_SECRET_KEY}`).
- **두 개의 Redis 키**(`redis.cache.key`, 환경별 suffix `session:dev` 등):
  - `token_session` — 토큰 1개당 1키. 로그아웃/갱신 시 삭제 → **서명이 유효해도 이 키가 없으면 인증 실패**(서버측 폐기).
  - `user_session` — 사용자당 1키(`company`+`no`). 인증 후 `c.Locals("User")`로 꺼내는 사용자 정보.
  - `user_fail` — 로그인 실패 카운터. TTL = `jwt.exp`.
- **`jwt.exp`는 환경별로 크게 다르다**: local `31536000s`(1년), dev `7200s`, live `1800s`(30분). 실패 카운터 TTL도 `jwt.exp`를 재사용하므로 live에서 "30분 후 재시도" 문구와 맞는다. exp를 바꾸면 **잠금 해제 시간도 같이 바뀐다.**
- **`RoleApi` 토큰은 Redis 세션을 보지 않는다.** `Authentication` 미들웨어는 토큰 역할에 `API`가 있으면 `token_session`/`user_session` 조회 없이 통과시킨다(`c.Locals("Role")`만 설정, `User` 없음). 즉 `API` 토큰 핸들러에서는 `c.Locals("User")`가 없다.

### 역할

- `types.Role`: `ADMIN`, `API` 등 (`aidea-api-common/types/role.go`).
- 일반 사용자 역할은 `config.role`(admin/tester 목록, `m:phddy` 형식 = `{company}:{id}`)에서 `MakeRole`로 계산한다. **관리자 지정은 YAML의 `role.admin` 목록**이다.
- `AIDEA_ACCESS_ROLE`(필수 env, 비면 부팅 panic) — 이 서비스/인스턴스에 접근 가능한 역할. `/auth/refresh` 등 일반 인증 라우트의 허용 역할이 된다. `/user`, `/api`는 여기에 `RoleApi`를 더해 허용한다.
- `/auth/make`는 `authMiddleRouter`(AccessRole)를 거친 뒤 `adminRole`(`ADMIN`)을 한 번 더 건다.

## 에러 처리

`errors/`가 이 서비스의 에러 정의와 중앙 핸들러를 가진다. 로컬 `errors` 패키지(`aidea-api-auth/errors`)와 `hexim/errors`(별칭 `er`)를 **함께** 쓴다.

| 상수 | 코드 | 용도 |
|------|------|------|
| `Authentication` | 40100 | 일반 인증 실패 |
| `IncorrectPassword` | 40101 | 비밀번호 불일치 |
| `UserIdNotFound` | 40102 | 사용자 없음 / DXE 미동의 |
| `TooManyFailedLogin` | 40103 | 3회 이상 실패 |
| `BedRequest` | 40000 | 잘못된 요청 (오타 그대로 `Bed`, 기존 이름 유지) |
| `ServiceUnavailable` | 50000 | 기본 서버 오류 |

- 코드는 `XXXYY` 5자리(HTTP 상태 = `code/100`). `code < 10000`이면 `*100` 보정.
- 응답 형식: `{id, code, message}` (go-conventions의 `{id, message, detail}`과 **다르다**. 이 서비스 `Handler()` 기준).
- **인증 관련 코드(40100~40103)는 로깅하지 않는다.** 그 외는 `zap` Error로 요청 id/ip/method/path/query/body(+user_no) 필드와 함께 남긴다. multipart는 body 대신 content-type만.
- 사용자 입력 오류와 사용자 없음을 **같은 `UserIdNotFound`로 뭉쳐서** 반환한다(계정 존재 여부 노출 최소화 의도). `userLogin`에서 쿼리 에러·ErrNoRows 모두 `UserIdNotFound`로 감싼다.

## 설정 / 환경

- `ENV`(`local`/`dev`/`live`, 기본 `local`) → `config/<env>.yaml`. `config/index.go`의 `init()`이 `hexim.LoadEnv()` + `.env` 로드 + YAML 파싱 + `AIDEA_ACCESS_ROLE` 검증을 수행한다(실패 시 `log.Panic`).
- YAML의 `${ENV_VAR}` 치환 값: `JWT_SECRET_KEY`, `MYSQL_USER`/`MYSQL_PASSWORD`(anxer, anxer_ro), `DATAMAX_MYSQL_USER`/`DATAMAX_MYSQL_PASSWORD`(datamax).
- **MySQL 3개**: `datamax`(사내 인사 DB, 외부 호스트, **읽기 전용**), `anxer`(쓰기, `user_login_log`), `anxer_ro`(읽기, 로그인 이력 조회).
- **Redis**: `cache`(Valkey, TLS). 키 3종(`token_session`/`user_session`/`user_fail`)은 YAML에 정의.
- 로그: 에러 로그 + 커스텀 `auth` 파일 로그(`logs/%Y%m%d_%IP%_auth.log`). 파일 로그 포맷은 `시각(yyyyMMddHHmmss) 구분(L|R) company userNo status` 순서의 구분자 행.
- 타임아웃(`timeout.read/write/idle`), `shutdown_timeout`, `cors` 목록은 YAML. graceful shutdown은 `shutdown_timeout-15`초 컨텍스트 + 10초 대기 후 `config.Close()`.

## 컨벤션과의 차이 (레거시)

이 레포는 `go-conventions.md`의 신규 규칙과 다르게 짜여 있다. **고칠 때 일관성을 먼저 보고, 범위를 벗어나는 대규모 리팩터는 별도로 제안한다.**

- 패키지 레벨 `var`에서 `cfg.Get()` → DB/Redis 클라이언트를 만든다(`router/auth/index.go`, `login.go`, `router/user`, `router/index.go`). `Router()` 내부 var 블록으로 조립하는 건 `router/api`뿐이다. 테스트 교체가 어렵다.
- 핸들러가 `func login(c fiber.Ctx) error` 형태의 **패키지 함수**다(`type api struct` 방식 아님).
- store/service 분리 없음. sqlboiler 쿼리와 raw SQL(`loginMezzoQuery`/`loginDxeQuery`)이 핸들러·login.go에 섞여 있다.
- `router/auth`의 `Router`가 `handlers ...any`를 받지만 `router/index.go`의 `Router`는 `...fiber.Handler`다(시그니처 불일치).
- `.env`, `go.work`, `sqlboiler*.toml`은 gitignore 대상(로컬 전용)이다. `logs/`는 `*.log`만 무시된다.

## 알려진 주의점 (작업 시 확인)

- **`/auth/make`의 `expired`는 현재 토큰 만료에 반영되지 않는다.** `JwtMake.UnmarshalJSON`이 `expired`(`YYYY-MM-DD`)를 파싱해 필수 필드로 요구하지만, `makeJwt`는 `jwtService.MakeToken(company, -1, role...)`을 호출해 만료는 항상 `jwt.exp`다. 또한 이 토큰은 `token_session`을 만들지 않으므로 `API` 역할이 아니면 인증이 통과하지 못한다(세션 키 없음). 이 동작을 바꾸려면 `MakeToken` 시그니처(common)와 소비 서비스를 함께 확인한다.
- **`/api/user/token`은 서버 간 호출 전용이다.** 호출 측은 `API` 역할 토큰(`/auth/make`로 발급)을 쓴다. 비밀번호 검증 없이 `no`만으로 사용자 토큰을 발급하므로 `API` 토큰 노출 = 임의 사용자 사칭이다. 이 토큰을 로그/응답에 남기지 않는다.
- **로컬은 비밀번호 검증이 꺼져 있다**(`IsLocal`). 로컬 동작을 근거로 live 로그인 동작을 판단하지 않는다.
- `GetIncr`는 Redis 에러를 0으로 삼킨다 → Redis 장애 시 실패 잠금이 **열린 채로(fail-open)** 동작한다.
- `search`는 결과가 없으면 `nil`을 반환해 **200 + 빈 본문**이 된다(204 아님, `openapi.yaml`과 맞는지 확인).
- 사용자 `like` 검색은 `"%"+입력+"%"` 파라미터 바인딩이라 SQL 인젝션은 아니지만, 인덱스를 타지 않는 전체 스캔이다. datamax는 외부 DB이므로 호출 빈도에 유의한다.
- `sqlboiler.toml`/`sqlboiler_datamax.toml`은 DB 접속 정보를 담는다. 값을 문서/로그/응답에 옮기지 않는다. 두 파일은 gitignore(`sqlboiler*.toml`) 대상이라 저장소엔 없고 로컬에만 있다(현재 `sqlboiler.toml`에 평문 자격 증명이 있으니 로컬 보관만 하고 공유하지 않는다).

## 데이터베이스 / 코드 생성

- `database/anxer`: `sqlboiler.toml` (`whitelist = ["user_login_log"]`, `login_status` enum `["0","1"]`, `no-tests = true`).
- `database/datamax`: `sqlboiler_datamax.toml`. `dmp_user_inner`, `dxe_user_inner`, 그리고 `user.go`(변환 헬퍼, **수기 작성**)가 같이 있다.
- 생성 코드(`boil_*.go`, `mysql_upsert.go`, 테이블 파일)는 직접 수정하지 않는다. `user.go`만 손으로 관리한다.
- 두 DB 모두 쿼리 스타일은 go-conventions의 타입드 query mod 대신 **문자열 `qm.Where("no = ?", ...)`**을 쓴다. 이 레포 안에서는 기존 방식에 맞춘다.
- 로그인 이력 조회는 `anxer_ro`(읽기), insert는 `anxer`(쓰기). 로그인 직후 방금 쓴 로그를 읽는 경로는 없다(복제 지연 이슈 없음).

## 빌드 / 배포

```bash
go run .          # ENV=local 기본, :5001
go build ./...    # 커밋 전 필수
go vet ./... && gofmt -l .
```

- 레포에 `*_test.go`가 **없다.** 기능 추가/버그 수정 시 테스트를 함께 추가한다. 외부(Redis/MySQL/datamax) 의존이 크므로 `aidea-api-spinx`처럼 통합 테스트 방식 + `hexim.LoadEnv()`가 현실적이다.
- **Dockerfile**: 멀티스테이지, `BUILD_TAG`(live는 `-tags production`), `config/*.yaml`·`openapi.yaml` 복사, `dumb-init` 엔트리포인트(prefork).
- **Jenkinsfile**: 병렬 아치 빌드 → Harbor(`aidea/aidea-api-auth`) → manifest → `argocd-apps` 이미지 태그 갱신. `argocd-apps/aidea/aidea-api-auth/{base,dev,stg,live}`(kustomize, live는 Rollout/HPA/PDB 패치).

## 자주 건드리는 지점

- 새 엔드포인트 → 해당 `router/<group>`에 추가, `Router()`에서 등록. 인증 범위(`AccessRole`/`ADMIN`/`API`)를 먼저 정하고 **openapi.yaml도 같이 갱신**한다.
- 로그인 정책(잠금 횟수/해제 시간/비밀번호 검증) → `login.go`의 `failCheck`/`failCount`/`userLogin`. TTL이 `jwt.exp`에 묶여 있음을 기억한다.
- 토큰/세션 구조 변경 → `makeToken`, `router/api`(발급 코드 **중복**), `aidea-api-common`의 `Authentication`·`JwtService`를 함께 본다. 다른 모든 서비스가 영향받는다.
- 사용자 조회 쿼리 변경 → `router/user`, `router/api`, `login.go`의 raw SQL, `database/datamax/user.go` 변환 헬퍼 4곳이 비슷한 로직을 각자 갖고 있다.
- 새 에러 → `errors/index.go`에 상수 추가. 인증 계열이면 `Handler()`의 로그 제외 `switch`에도 코드를 넣는다.
- 공용 라이브러리(`aidea-api-common`, `hexim`) 수정 시 이 레포도 `go build ./...`로 함께 검증한다(`go.work`로 묶임).
