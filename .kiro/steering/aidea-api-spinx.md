---
inclusion: fileMatch
fileMatchPattern: 'aidea-api-spinx/**'
---

# AIDEA-API-SPINX

CJ MezzoMedia AI 챗봇 **앤써(AnXer)**의 외부 연동 API 게이트웨이. Anthropic/OpenAI/Google/Bedrock 등 멀티 프로바이더 LLM을 단일 인터페이스(`/spinx/{provider}`)로 호출하고, 세션 기반 대화(요약 포함)·스트리밍(SSE)·툴/Tool Choice·MCP 연동(승인 워크플로우)·RAG(OpenSearch)·사용량 관리를 제공한다.

- **모듈 경로**: `github.com/mz-team-ax/aidea-api-spinx`
- **Go 버전**: 1.26 (Alpine 3.23)
- **경로**: `/Users/phddy/Projects/dx/aidea-api-spinx`
- **HTTP 포트**: `:5003`
- **기능 상세**: 엔드포인트·프로바이더·환경변수 등은 레포 `README.md`가 1차 소스. 이 문서는 **코드 구조와 작업 시 따를 컨벤션**에 집중한다.

> 공통 Go 컨벤션(DI, 에러, 로깅, SQLBoiler, 테스트 등)은 `go-conventions.md`를 따른다. 여기서는 spinx에 고유한 구조·패턴만 다룬다.

## 레이어 구조

요청 흐름은 `router → service → store/외부시스템` 이다. 레이어를 역할로 엄격히 가른다.

- **`router/`** — Fiber 핸들러. `type api struct`에 의존성을 담고 핸들러는 `(a *api)` 메서드. `Router()`가 조립 지점(아래 "라우터 DI" 참고).
  - `router/spinx` — 채팅/세션/툴/RAG/앱별 MCP/스트리밍
  - `router/mcp` — 관리자용 MCP 승인·거부·도구 조회
  - `router/usage` — 사용량 조회·집계·리포트
- **`service/`** — **오케스트레이션 계층**. LLM 호출(`hexim/llm`), Redis 캐시, OpenSearch, Kiwi(gRPC), 메일 등 외부 시스템 연동과 비즈니스 로직. DB 접근은 직접 하지 않고 `store`에 위임한다.
- **`store/`** — **DB 전용 계층**(SQLBoiler). `store.Session`/`Usage`/`MCP`/`RAG`, 생성자 `store.NewXxx(db, readDb *sql.DB)`. 비즈니스 로직 없음. 읽기 `s.readDb`, 쓰기 `s.db`.
- **`database/`** — SQLBoiler 생성 코드. 직접 수정 금지.
- **`types/`** — 모델 alias, MCP/RAG/사용량 도메인 타입 + 변환 헬퍼.
- **`cmd/<name>/`** — 주기 배치(`mcp-checker`, `mcp-reviewer`). ArgoCD cron으로 스케줄. 아래 "MCP 배치" 참고.

> **판단 기준**: `s.db`/`s.readDb`로 sqlboiler만 두드리면 `store`, LLM/Redis/OpenSearch 등 외부 클라이언트를 들고 조합하면 `service`. "DB만 다루는 코드를 service라 부르지 않는다."

### store ↔ service 분리 시 주의

- **Redis/OpenSearch/Kiwi/LLM은 store가 아니다.** 세션 요약 캐시(Redis), RAG 검색(OpenSearch), 형태소 분석(Kiwi), 임베딩/요약(LLM)은 전부 `service`에 남긴다. store는 MySQL만 안다.
- service는 cache-aside를 직접 조립한다: Redis 조회 → miss면 store 조회 → Redis 저장. (`service.Session`의 `loadMessages`/`Summary` 참고)
- store 생성자는 에러를 낼 조립 로직이 없으면 `*Store`만 반환한다(예: `NewSession(db, readDb) *Session`). 외부 연결을 들면 `(*T, error)`.

## 트랜잭션 + 외부호출 원자성 (중요)

여러 DB 쓰기나 "DB 쓰기 + 외부 시스템 작업"이 원자적이어야 하면, **store가 트랜잭션을 소유하고 외부 작업은 콜백으로 주입**받는다. 트랜잭션(`*sql.Tx`)을 store 밖으로 노출하지 않는다. LLM/OpenSearch 의존은 service에 남긴 채, "그 작업이 필요하면 이 함수를 불러라"만 store에 넘긴다.

적용 사례(그대로 따를 것):

- **`store.MCP.Review(ctx, serverNo, embed func(name, desc string, schema []byte) (hash string, embedding []byte, err error))`**
  store가 tx를 열고 → tool들 조회 → 각 tool마다 `embed` 콜백(= service의 LLM 임베딩) 호출 → tool 업데이트 → 서버 상태 전환까지 한 트랜잭션. 임베딩 하나라도 실패하면 전부 롤백.
- **`store.RAG.InsertIndex(ctx, appNo, name, createIndex func(ragNo int64) error)`** / **`DeleteIndex(ctx, appNo, ragNo, deleteIndex func() error)`**
  MySQL insert/delete를 tx에 쌓고, 콜백에서 OpenSearch 인덱스를 생성/삭제. 콜백 실패 시 tx 롤백 → MySQL에 고아 레코드가 남지 않는다. **커밋은 콜백 성공 후에만.**
- **`store.MCP.SaveReview(ctx, server, toolReviews, review)`**
  AIReview의 tool 판정 + 서버 상태/설명/임베딩을 한 트랜잭션에 일괄 저장.

### 알아둘 한계

- 콜백(LLM/OpenSearch = 네트워크)이 **트랜잭션 안에서** 실행되므로 트랜잭션이 길어진다. 원자성과 맞바꾼 의도된 선택이다. 대상이 수백 건이면 트랜잭션 점유 시간을 고려할 것.
- 콜백 성공 후 `tx.Commit()` 자체가 실패하는 극히 드문 경우엔 역방향 불일치(예: OpenSearch엔 있고 MySQL엔 없음)가 가능하다. 완전한 2PC는 불가. 삭제 콜백은 **멱등**하게(이미 없으면 404를 성공 취급) 만들어 재시도가 수렴하게 한다.

## MCP 상태 비트 쓰기 (동시성)

`spinx_mcp_server.status`는 비트플래그(`types/mcp_status.go`)다. `suspended`/`disconnected`/`changed`처럼 **개별 비트를 켜고 끄는 쓰기는 read-modify-write로 하지 않는다.** 앱의 5분 ticker(`initMCPServers`)와 `cmd/mcp-checker` 배치가 같은 서버 row를 동시에 건드릴 수 있어, "읽어서 Go에서 비트 조작 후 전체 status 덮어쓰기"는 상대가 세운 다른 비트를 날리는 lost update를 만든다.

- **비트 누적/해제는 UPDATE 안에서 원자적으로.** `status = status | ?`(켜기), `status = status & ~?`(끄기)를 `queries.Raw(...).ExecContext`로 DB에 위임한다. `failure_count = failure_count + 1` 같은 증가도 동일.
  - 적용: `store.MCP`의 `SetDisconnected`/`SetPingFailed`/`SetPingSucceeded`/`MarkChanged`.
- **"이미 켜져 있었나" 판정이 필요하면 WHERE 가드 + affected rows.** 예: `MarkChanged`는 `WHERE no = ? AND status & <changed> = 0`으로 UPDATE하고 `RowsAffected()`가 1이면 이번에 세운 것(= 메일 발송), 0이면 이미 켜져 있던 것(= skip). 멱등성과 원자성을 동시에 얻는다.
- **예외**: `Approve`/`Reject`는 비트 누적이 아니라 라이프사이클 상태를 **마스크로 통째 세팅**(`status.Approve()` 등)하고, 사람이 트리거하는 저빈도 작업이라 read-modify-write(`findServer` → `updateServerStatus`)를 유지한다. ping/`changed`처럼 배치와 경합하는 고빈도 비트 조작과는 성격이 다르다.

## MCP 배치 (cmd)

무거운 주기 작업은 HTTP 핸들러가 아니라 `cmd/<name>/main.go` 배치로 분리하고, ArgoCD cron으로 스케줄한다. `main`이 조립 지점(`init()`/전역 var 금지, go-conventions 참고), 동시성은 `errgroup.SetLimit(10)`으로 제한, 실제 에러/panic일 때만 `os.Exit(1)`.

- **`mcp-checker`** — approved 서버의 헬스체크. 서버마다 Connect→Ping→tool 재동기화(`SetTools`). 실패 단계별로 `SetDisconnected`/`SetPingFailed`(3회 초과 시 suspended)/`SetPingSucceeded`. ping 실패는 인프라 에러이므로 exit 1 대상.
- **`mcp-reviewer`** — `requested` 서버(= AI 심사 대기 큐)를 긁어 각각 `AIReview` 실행. **tool이 rejected로 판정되는 건 정상 결과**(AIReview가 `nil` 반환)이므로 exit 0. LLM/DB/파싱 실패나 panic만 exit 1. 조회는 `store.MCP.RequestedServers`.

**ArgoCD cron 스케줄** (`argocd-apps/aidea/aidea-api-spinx/base/cronjob.yaml`)

- `mcp-checker` (`aidea-api-spinx-mcp-cronjob`): `*/5 * * * *` — 5분마다 상시. `activeDeadlineSeconds: 270`.
- `mcp-reviewer` (`aidea-api-spinx-mcp-reviewer-cronjob`): `2,32 8-17 * * 1-5` + `timeZone: "Asia/Seoul"` — 평일 08:02~17:32, 30분 간격(하루 20회), 18시 이후·주말·야간 미실행. checker(`*/5`)와 DB/LLM 동시 타격을 피하려 2분 오프셋. `activeDeadlineSeconds: 900`(LLM 심사 몰림 대비).
- 공통: `concurrencyPolicy: Forbid`(이전 실행 미완 시 스킵 → 백프레셔), `backoffLimit: 0` + `restartPolicy: Never`(재시도 없음 — 상태가 곧 큐라 다음 주기에 다시 집힘). 리뷰 주기를 짧게 둘 실익이 작은 이유: `ai_reviewed` 다음 단계가 **관리자 수동 /approve**라 전체 승인 리드타임의 병목은 사람이지 배치 주기가 아니다.

## 라우터 DI

`init()` + 패키지 전역 var를 쓰지 않는다. 핸들러 의존성은 `type api struct`에 모으고 `Router()`에서 조립한다(공통 규칙은 go-conventions의 "HTTP 핸들러 DI" 참고). spinx의 세 라우터(`spinx`/`mcp`/`usage`) 모두 이 방식으로 전환 완료.

- `Router()` 안에서 `var (...)` 블록으로 config/clients/store/service를 조립 → `a := &api{...}` → 핸들러 등록.
- `hexim/llm`의 `Store()`/`MCPStore()`/`ToolStore()`는 프레임워크 전역 싱글톤이라 `Router()`에서 받아 `api` 필드로 주입한다.
- 부팅 로직(예: `router/spinx`의 MCP 자동연결 + 5분 주기 ticker)은 `init()`이 아니라 `Router()`에서 `a.initMCPServers(...)` 메서드로 실행한다. `fiber.IsChild() || config.IsLocal` 조건 유지.
- `api`에는 **실제 핸들러가 읽는 의존성만** 둔다(쓰지 않는 필드 주입 금지).
- 조립 실패(DB/필수 설정)는 `Router()`에서 `log.Panic`. 현재 각 `Router()`가 `cfg.Get()`로 자급자족하는 건 레거시 잔재이며, 장기적으로 `main` 주입이 이상형(go-conventions 참고).

## MCP 상태 전환

서버 상태 전환은 **의도별 전용 메서드**로 분리한다(과거 거대 `SetServerStatus` switch는 폐기). 각 메서드는 DB 전환 + 필요 시 메일 발송을 조립한다.

- `service.MCP.Approve(serverNo)` — ai_reviewed 상태에서만 승인(가드 포함) + 승인 메일
- `service.MCP.Reject(serverNo, reason)` — 거부 + 사유 기록 + 거부 메일
- `service.MCP.MarkChanged(serverNo)` — changed 플래그(멱등: 이미 있으면 write/메일 skip)
- `service.MCP.Review(serverNo)` — requested 전환 + tool 임베딩(위 콜백 트랜잭션)
- `service.MCP.AIReview(serverNo)` — LLM 보안 심사 + 서버 설명/임베딩 생성 후 `SaveReview`로 ai_reviewed 전환. **핸들러가 직접 호출하지 않고 `cmd/mcp-reviewer` 배치가 호출**한다(아래 "MCP 배치" 참고).
- `MailService`는 `service.MCP`에 생성자 주입. `sendMail`은 `(m *MCP)` 메서드.

**상태 흐름**: `pending → (앱 /review) requested → (mcp-reviewer 배치의 AIReview) ai_reviewed → (관리자 /approve) approved`, 언제든 `rejected`/`suspended`. `AIReview`의 실패/panic은 `defer`에서 **서버만** rejected로 저장(부분 tool 저장 금지 — `SaveReview`에 `nil` 전달).

> **핸들러는 큐잉만, 전환은 배치가 한다.** `/spinx/mcp/:id/review`(`requestMCPReview`)는 `service.Review`로 상태를 `requested`로만 올리고 끝낸다(무거운 LLM 심사를 HTTP 요청에서 동기로 돌리지 않는다). 실제 `requested → ai_reviewed` 전환은 `mcp-reviewer` 배치가 `requested` 서버를 긁어 `AIReview`를 돌려 수행한다. 상태가 곧 큐이므로 배치 재시작/배포에도 유실 없이 다음 주기에 다시 집힌다.

## 테스트

- 이 레포 테스트는 전부 **실제 외부 의존에 붙는 통합 테스트**다(MySQL/Redis/OpenSearch/Bedrock/Kiwi). mock을 쓰지 않는다. `hexim.LoadEnv()`로 로컬 설정을 로드한다.
- **store 전용 테스트를 따로 만들지 않는다.** service 통합 테스트가 store 경로까지 커버한다. 대신 "콜백 실패 시 롤백" 같은 트랜잭션 로직은 service 테스트에서 검증한다.
- 실제 리소스를 만드는 테스트(RAG 인덱스 등)는 **`t.Cleanup`으로 정리를 보장**한다. 서브테스트 간 공유 상태의 정리는 부모 테스트의 `t.Cleanup`에 등록한다(서브테스트 `t`에 등록하면 해당 서브테스트 종료 즉시 실행되어 뒤 단계가 깨진다).
- 외부 연결(실 MCP 서버 등)이 필요한 테스트는 `*_Live` 네이밍 + `testing.Short()` skip으로 의도적으로만 실행한다.
- Kiwi(기본 `:5100`) 등 필수 외부 서비스가 없으면 **skip이 아니라 에러로** 실패시킨다(환경이 갖춰져야 통과).

## 자주 건드리는 지점

- 새 DB 접근 → `store/`에 메서드 추가(타입드 query mod, `er.Wrap`). service는 위임.
- 새 핸들러 → 해당 `router/<group>`의 `(a *api)` 메서드로. 의존성은 `api`에 추가하고 `Router()`에서 주입.
- DB + 외부시스템 원자성 필요 → store에 콜백 받는 tx 메서드 추가(위 패턴).
- MCP status 비트 켜기/끄기 → read-modify-write 금지. `queries.Raw`로 `status = status | ?` / `& ~?` 원자 UPDATE(위 "MCP 상태 비트 쓰기").
- 무거운 주기 작업 → 핸들러 말고 `cmd/<name>` 배치로. 상태를 큐로 쓰고(예: `requested`) 배치가 집어가게.
- 커밋 전 `go build ./...`, `go vet ./...`, `gofmt -l`. 통합 테스트는 로컬 외부 서비스가 떠 있을 때 `go test ./service/... ./store/...`.
