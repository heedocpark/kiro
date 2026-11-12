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

- **`store.MCP.SyncTools(ctx, connNo, serverNo, remote)`**
  원격 tool 목록을 한 트랜잭션에 반영: tool 내용 버전 찾기/생성 → connection_tool 포인터 갱신 → `last_seen_at`/deleted 일괄 UPDATE. 중간 실패 시 전부 롤백.
- **`store.RAG.InsertIndex(ctx, appNo, name, createIndex func(ragNo int64) error)`** / **`DeleteIndex(ctx, appNo, ragNo, deleteIndex func() error)`**
  MySQL insert/delete를 tx에 쌓고, 콜백에서 OpenSearch 인덱스를 생성/삭제. 콜백 실패 시 tx 롤백 → MySQL에 고아 레코드가 남지 않는다. **커밋은 콜백 성공 후에만.**
- **`store.MCP.SaveAIReview(ctx, serverNo, verdicts, desc)`**
  AI 판정(tool 버전별 verdict/임베딩) + 서버 설명 + `requested → ai_reviewed` 전이를 한 트랜잭션에. 서버 전이 가드(affected 1)가 실패하면(그사이 관리자 reject 등) 전부 롤백. LLM/임베딩 호출은 service에서 트랜잭션 **밖**에서 끝낸 뒤 결과만 넘긴다.
### 알아둘 한계

- 콜백(LLM/OpenSearch = 네트워크)이 **트랜잭션 안에서** 실행되므로 트랜잭션이 길어진다. 원자성과 맞바꾼 의도된 선택이다. 대상이 수백 건이면 트랜잭션 점유 시간을 고려할 것.
- 콜백 성공 후 `tx.Commit()` 자체가 실패하는 극히 드문 경우엔 역방향 불일치(예: OpenSearch엔 있고 MySQL엔 없음)가 가능하다. 완전한 2PC는 불가. 삭제 콜백은 **멱등**하게(이미 없으면 404를 성공 취급) 만들어 재시도가 수렴하게 한다.

## MCP 데이터 모델과 상태 쓰기 (동시성)
기조: **MCP는 심사(AI 1차 + 관리자 확정)를 받아야 서비스되고, 심사 후 변경된 것은 재심사로 처리한다.** 상세는 레포 `docs/mcp-overview.md`.

- **테이블 4개, 심사 대상과 사용 주체 분리**: `spinx_mcp_server`(URL당 1행, 심사 대상) / `spinx_mcp_server_tool`(`(server, name, hash)`당 1행, 심사 단위 = tool 내용 버전) / `spinx_mcp_connection`(`(app, server)`당 1행, 토큰) / `spinx_mcp_connection_tool`(토큰별로 보이는 tool → 현재 버전 포인터). 같은 URL·같은 내용은 한 번 심사하면 모든 앱이 공유한다.
- **상태는 축마다 컬럼 하나, 컬럼마다 쓰는 주체 하나.** 비트플래그 금지. 값은 `types/mcp_status.go`의 Go 상수로만 검증(DB CHECK 없음).
  - server `approval_status`(pending/approved/rejected, 관리자) · `review_status`(none/changed/requested/ai_reviewed, 앱·배치·관리자)
  - tool 버전 `ai_verdict`(AI) · `approval_status`(관리자 확정)
  - connection `connection_status`(ok/disconnected/suspended, checker)
  - connection_tool `selection_status`(앱) · `remote_status`(present/deleted, 동기화)
- **노출 조건은 store 한 곳에서만 판단**(`ServableTools`/`AvailableConnections`): server approved + connection ok + selected + present + 버전 approved + embedding 있음. 라우터에서 필드를 조합해 판단하지 않는다.
- **내용 변경 = 새 버전.** tool hash는 hexim이 원격 원본으로 계산한 값을 그대로 저장하고 DB 값으로 재계산하지 않는다(JSON 컬럼 정규화로 hash가 달라져 무한 재심사가 된다). 버전 행은 불변이고 connection_tool의 `tool_no`만 옮긴다.
- **경합하는 고빈도 전이는 가드 UPDATE + affected rows.** `UPDATE ... SET review_status='changed' WHERE no=? AND approval_status='approved' AND review_status='none'`처럼 이전 상태를 WHERE에 건다. affected 1이면 이번에 바꾼 것(메일 1회), 0이면 skip. 적용: `RequestReview`/`MarkChanged`/`SaveAIReview`/`FailAIReview`, 연결 상태(`SetDisconnected`/`SetPingFailed`/`SetPingSucceeded`).
  - MySQL `SET`은 왼쪽부터 평가되어 앞 대입 결과가 뒤 식에 보인다. 원래 값을 봐야 하는 식을 앞에 둔다(`FailAIReview`, `SetDisconnected` 참고).
- **관리자 결정(Approve/Reject)은 트랜잭션 + `SELECT ... FOR UPDATE`.** 서버 행을 잠그고 tool 버전 확정 등 여러 행을 함께 바꾼다. 저빈도라 잠금 비용은 문제없다.
- **SQL에서 시간 비교에 쓰는 컬럼은 DB `NOW()`로만 쓴다**(`last_seen_at` 삭제 유예 15분). Go `time.Now()`와 섞으면 앱/DB 시간대 차이로 판정이 틀어진다. 표시용 시각은 Go 시간이어도 된다.
- **JOIN 쿼리에서는 `XxxTableColumns`(테이블 한정 컬럼)를 쓴다.** `XxxColumns`는 비한정이라 `no`/`name`/`created_at`이 겹치면 ambiguous 에러. where 헬퍼는 생성 코드가 이미 한정한다.
- 목록 조회는 `qm.Select`로 `embedding`을 제외한다(수십 KB).
- 이름 컬럼(`name`, `url`)은 `utf8mb4_bin` — MCP tool 이름은 대소문자를 구분한다.

## MCP 배치 (cmd)

무거운 주기 작업은 HTTP 핸들러가 아니라 `cmd/<name>/main.go` 배치로 분리하고, ArgoCD cron으로 스케줄한다. `main`이 조립 지점(`init()`/전역 var 금지, go-conventions 참고), 동시성은 `errgroup.SetLimit(10)`으로 제한, 실제 에러/panic일 때만 `os.Exit(1)`.

- **`mcp-checker`** — approved 서버의 **connection**(app + 토큰)마다 헬스체크. Connect→Ping→tool 동기화(`SyncConnection` → `store.SyncTools`). 실패 단계별로 `SetDisconnected`/`SetPingFailed`(3회 초과 시 suspended)/`SetPingSucceeded`. suspended도 대상에 포함해 복구를 감지한다. ping 실패는 인프라 에러이므로 exit 1 대상.
- **`mcp-reviewer`** — `review_status=requested` 서버(= AI 심사 대기 큐)를 긁어 각각 `AIReview` 실행. **tool이 rejected로 판정되는 건 정상 결과**(AIReview가 `nil` 반환)이므로 exit 0. LLM/DB/파싱 실패나 panic만 exit 1. 조회는 `store.MCP.RequestedServers`.

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
서버 상태 전환은 **의도별 전용 메서드**로 분리한다. 각 메서드는 DB 전환 + 필요 시 메일 발송을 조립한다. 서버 공용 상태 메일은 그 서버를 쓰는 모든 connection의 `notify_email`로(`notifyServer`).
- `service.MCP.RequestReview(app, connNo, dryRun)` — none/changed → requested. 심사 대상(selected + present + pending 버전)이 없으면 400(rejected 서버는 서버 단위 재심사 허용). 무거운 작업 없이 큐잉만.
- `service.MCP.AIReview(serverNo)` — LLM 보안 심사(20개 배치, 배치별 no 매핑, 누락 시 실패) → AI 승인 버전만 임베딩 → 승인 tool 집합 hash가 바뀌면 서버 설명 재생성 → `SaveAIReview`. **핸들러가 직접 호출하지 않고 `cmd/mcp-reviewer` 배치가 호출**한다. 실패/panic은 `defer`에서 `FailAIReview`(tool 판정 저장 안 함): 최초 심사는 rejected, 재심사는 approved 유지 + changed.
- `service.MCP.Approve(serverNo)` — ai_reviewed에서만. tool 버전 `ai_verdict` → `approval_status` 확정, 서버 approved. 심사 중 새 변경이 남아 있으면 review를 changed로.
- `service.MCP.Reject(serverNo, reason)` — 심사 중이면 이번 AI 판정 무효화(최초: rejected / 재심사: approved 유지 + changed), 심사 중이 아니면 **승인 철회**(kill switch).
- `markChanged(serverNo)` — 동기화/선택 후 "selected + present + pending 버전"이 있으면 호출. approved 서버만 none → changed, 이번에 바뀐 경우만 메일.
- `MailService`는 `service.MCP`에 생성자 주입. 메일은 `context.WithoutCancel`로 요청 종료와 무관하게 보낸다.

**상태 흐름**: `review_status` `none → (앱 /review) requested → (mcp-reviewer AIReview) ai_reviewed → (관리자 /approve) none`, 이때 `approval_status` approved. 승인 후 변경 감지 시 `changed` → 앱이 `/review`로 재심사 요청(자동 재심사 안 함 — 어차피 관리자 승인이 필요). 재심사 중에도 기존 승인분은 계속 서비스.

> **핸들러는 큐잉만, 전환은 배치가 한다.** `/spinx/mcp/:id/review`는 상태를 `requested`로만 올리고 끝낸다. 상태가 곧 큐이므로 배치 재시작/배포에도 유실 없이 다음 주기에 다시 집힌다.

**앱 API의 `:server_id`는 connection id**, 관리자 API(`/mcp/:server_id`)는 server id다. 채팅 요청의 `mcp.servers[].id`(connection id)는 `filterMCPServers`로 앱 자신 + global의 사용 가능한 connection만 허용한다(다른 앱 토큰 사용 방지). 승인 후 URL 변경은 불가(새로 등록), 헤더(토큰)는 자유 수정.

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
- MCP 상태 전이 → 컬럼별 가드 UPDATE(`WHERE <이전 상태>` + affected rows) 또는 관리자 결정은 tx + `FOR UPDATE`. 비트 연산·read-modify-write 금지(위 "MCP 데이터 모델과 상태 쓰기").
- MCP 노출 조건 변경 → `store.ServableTools`/`AvailableConnections` 한 곳만 고친다.
- 무거운 주기 작업 → 핸들러 말고 `cmd/<name>` 배치로. 상태를 큐로 쓰고(예: `requested`) 배치가 집어가게.
- 커밋 전 `go build ./...`, `go vet ./...`, `gofmt -l`. 통합 테스트는 로컬 외부 서비스가 떠 있을 때 `go test ./service/... ./store/...`.

## 토큰 사용량 기록 규칙
- hexim `Usage.InputToken`은 캐시 제외 값이다. spinx도 **`tokens`/`input_tokens`는 캐시 제외**, 캐시는 `cache_read_tokens`/`cache_write_tokens` 별도 컬럼에 저장한다(`spinx_message`·`spinx_session`·`spinx_daily_stat`). 집계 시 `tokens`에 캐시를 다시 더하지 않는다.
- user 메시지 행에 input + 캐시 read/write, assistant 행에 output만 넣는다(`router/spinx/index.go` chat/stream, `service/session.go` 요약 공통).
- Redis 일일 한도(`SetDailyUsageToken`)는 `InputToken + OutputToken`(캐시 제외).
- `anxer-tools`는 전역 `ToolStore`의 `*llm.Tool`을 **복사해서** 요청별 `Headers`/`Cache`를 넣는다(공유 객체 직접 수정 금지 — 동시 요청 간 headers 혼입).
- 요청의 `Content.Cache`/`Tool.Cache`는 `*llm.Request` 바인딩으로 hexim까지 그대로 전달된다. 사용 규칙은 `openapi.yaml` `chat_request`에 문서화.
