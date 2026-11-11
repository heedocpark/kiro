---
inclusion: fileMatch
fileMatchPattern: 'aidea-api-agent/{agent,cmd/workflow,cmd/agent,router/agent}/**'
---

# AIDEA-API-AGENT — 2계층 오케스트레이션 설계와 토큰/품질 교훈

> `agent/` 내부 구현(Worker/Verifier/Runner/Orchestrator/Coordinator/Recorder) 자체는 `aidea-api-agent-engine.md`를 본다. 이 파일은 그 위에 조립된 "워크플로우용 2계층 구조"의 설계 배경과, 실전 Live 디버깅으로 얻은 교훈을 담는다. 진입점(`cmd/workflow`, `cmd/agent`, `router/agent`) 또는 엔진을 함께 건드릴 때 로드된다.

## 세 진입점의 엔진 조립 비교

세 바이너리 모두 `agent/` 엔진을 쓰지만 **조립 방식과 전달하는 입력이 다르다.** 특히 `Input`의 `Upstreams`/`RunInput` 전달 여부가 다르다.

| 항목 | API `invoke` (`router/agent/run.go`) | agent CLI (`cmd/agent/service.go`) | workflow CLI (`cmd/workflow/service.go`) |
|---|---|---|---|
| 조립 | 단일 Runner (Worker + 조건부 Verifier) 고정 | 단일 Runner 고정 | **2계층 Coordinator (아래 참고)** |
| `Input.Upstreams` | ✗ 미전달 | ✗ 미전달 | ✓ `Load()`로 선행 task 결과 |
| `Input.RunInput` | ✗ 미전달 (`Input{Task, Goal}`만) | ✗ 미전달 | ✓ `s.run.UserInput` |
| 결과 저장 | `agentStore.SetRun` 직접 DB | `agent` API `/-/run` 호출 | `agent` API `/-/run` + output 파일 |

> **API invoke는 의도적으로 단일 Runner만 돈다.** 과거 `request.Orchestrator` 플래그로 Coordinator를 태우던 분기가 있었으나 제거했다 — 2계층 오케스트레이션은 (a) 즉석 동기 HTTP 요청엔 비용이 과하고(이중 Aggregate 등), (b) invoke는 단계 의존/upstream이 없어 stage 레이어가 빌 수밖에 없으며, (c) 호출자가 전략 적합성을 판단할 근거가 없기 때문. `cmd/agent`도 단일 Runner. **2계층 구조는 `cmd/workflow/service.go`에만 적용**된다. 오케스트레이션을 다른 진입점에도 열려면 요청 플래그가 아니라 **에이전트 버전 설정**으로 받는 방향이 맞다(남은 과제).

## 왜 2계층인가 — 문제와 설계

여행 워크플로우(호텔 조사 → 비교·추천)에서 **단일 Worker가 도구를 과다 호출해 입력 컨텍스트가 모델 윈도우를 초과**하는 문제가 있었다. 분할(Orchestrator/Coordinator)은 원래 "독립적인 일을 병렬로"가 전제인데, 실제 작업은 "발굴→조사→추천"처럼 순차 의존이 많고, 그 안의 "조사" 단계만 "호텔별로" 병렬화할 여지가 있다. 이 두 성격(순차 단계 vs 대상별 병렬)을 하나의 Plan으로 섞으면 Plan이 순번 범위("1~3번째 후보")로 어색하게 나누거나 지시서를 재작성해 원본을 손상시켰다. 그래서 역할을 레이어로 분리했다.

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
- **두 Plan은 역할이 다르다.** 바깥(Orchestrator Plan)은 순차 단계 골격만(도구 없음, "대상별 병렬은 그 단계 실행자가 알아서 한다"), 안쪽(Worker Plan)은 검색으로 대상 파악 후 대상별 병렬 분할(도구 있음).
- `NewCoordinator`가 `Runner` 인터페이스를 구현하므로 `NewRunner` 팩토리에 중첩 Coordinator를 끼우는 **조립**만으로 2계층이 된다. 재귀 호출이 아니라 조립 시점에 계층을 고정하므로 무한 분할 루프가 구조적으로 불가능하다(depth 카운터 불필요 — 안쪽 팩토리가 일반 Worker만 반환).
- 구체적인 조립은 `cmd/workflow/service.go`의 `Run` 메서드를 본다.

## 토큰/품질 디버깅에서 얻은 핵심 교훈 (실전 Live 로그 기반)

1. **tool_use + response_format(strict JSON)은 충돌한다.** 도구를 쓰는 LLM 호출에 strict JSON 스키마를 걸면, 모델이 멀티턴 중 형식을 맞추려 **필드를 빈 값으로** 채운다.
   - Worker는 이 이유로 response_format을 뺐다(커밋 `9fd8ad9`, `//ResponseFormat: workerResponseFormat()`).
   - **fanout Plan(도구 사용)도 같은 문제** → goal이 `""`로 반복 생성되어 `validatePlan`이 거부 → 재시도 소진 → 전체 실패. Plan이 sonnet인데도 발생(= 모델 역량 아님, 구조 문제). 처음엔 "haiku 역량 부족"이나 "재시도 피드백 부족"으로 잘못 짚었다가, Plan 모델이 sonnet이라는 사실을 확인하고서야 구조적 원인(도구+strict format 충돌)으로 좁혔다.
   - **해결**: `planRequest`가 `usesTools()`(MCP 활성)면 response_format 생략. 도구 없는 stage Plan은 유지. 안전망은 프롬프트 출력 안내 + `parsePlan`/`validatePlan`.
   - **디버깅 교훈**: 같은 모델이 간단한 걸 반복 실패하면 "모델 역량"보다 "구조적 제약(도구+format 충돌 등)"을 먼저 의심한다.

2. **Worker가 pass:false면 Verifier를 안 거친다.** 그래서 "Verifier가 타협 가이드를 준다"는 개선은 **Worker가 pass:true로 올려줄 때만** 작동한다. haiku Worker가 "확정가(미래 날짜 실시간가) 못 찾음"을 스스로 실패로 판정해 재시도를 소진(62K 토큰)하던 문제가 있었다. Verifier만 고쳐서는 이 경로를 못 잡는다 → **Worker 프롬프트의 Goal 타협 섹션**에 "검색으로 못 얻는 값은 즉시 '확인 안 됨' 타협 + pass:true. 핵심 항목 미확인만으로 pass:false 하지 마라. 그 판단은 Verifier가 한다"를 명시해 Worker 자체가 Verifier에 도달하게 했다.

3. **검증은 결과를 보고 과정은 보지 않는다.** Verifier/Plan이 "출처 N곳 이상 시도" 같은 **절차 요구**를 검증 조건으로 삼아 reject → 재시도 폭증. → Verifier 프롬프트에 "절차 요구는 검증 대상 아님, 결과물만 판정" 명시, Plan goal 작성 규칙에 "절차는 task에만, goal엔 결과 조건만" 명시.

4. **역할별 모델 분리.** Plan·Verifier = sonnet, 단순 조사 Worker = haiku(DB `version.MaxTokens`/모델 설정). 조사 Worker가 호출 수가 가장 많아 haiku로 비용 절감. 단 haiku는 JSON 형식을 자주 깨뜨린다(`invalid character 'ì'`/`'\n'`) — 아래 5번으로 구제.

5. **JSON 파서 관용성 강화** (`parse.go:jsonObject`). response_format 못 쓰는 Worker가 깨진 JSON을 뱉는 걸 구제: 첫 `{`부터 **중괄호 균형**이 맞는 `}`까지만 추출(앞뒤 설명 제거, 중첩 중괄호 정확히), **문자열 리터럴 안 생 제어문자**(줄바꿈 등)를 이스케이프. 세 파서(Worker/Aggregate/Verdict) 모두 거친다.

6. **`validatePlan`은 빈 task/goal을 모아서 한 번에 거부** — 재시도 때 모델이 전부 한 번에 고치도록. (단, 1번처럼 response_format이 원인이면 재시도로도 안 고쳐짐 → response_format 제거가 근본 해법. 프롬프트/피드백 강화만으로는 구조적 원인을 못 이긴다.)

## 프롬프트 캐시 배치 (`llm.WithCache`)
- 원칙: breakpoint는 **요청 간 동일한 구간의 마지막 블록**에만 건다. 그 앞 전체(tools → system → messages)가 캐시되고, 바뀌는 내용(`request`/`upstreams`/`fail_reasons`/`result`/`subtask_results`)은 반드시 뒤에 둔다. 최대 4개.
- 현재 배치 (5m):
  - **worker** `messages`: `user_instructions`(system+사용자 지시 — 같은 실행의 모든 subtask·실행 간 공유), `goal`(subtask 재시도)
  - **verifier** `messages`: `system`(모든 subtask 검증 공유), `goal`(검증 재시도)
  - **chat** `Stream`: `user_instructions`(대화 간 공유)
  - **orchestrator**: **캐시 없음**. Plan/Aggregate는 system prompt가 달라 서로 공유 안 되고 실행당 1회씩이라, 재시도가 없으면 write 할증(1.25x)만 남는다. 실측(2026-11-11)에서 orchestrator로 보이는 sonnet 호출 8건 모두 read 0 → 제거.
- 실측 요약(같은 run): haiku(worker) 히트율 60%·입력비용 −53%, sonnet(verifier 등) 히트율 32%·−25%. verifier의 system read(6936)는 매 호출 적중, goal breakpoint도 재시도에서 실제 재사용 확인.
- 남은 과제
  - **MCP 사용 호출**: `CandidateTools`가 요청 임베딩으로 도구를 골라 조합이 달라지면 prefix 맨 앞(tools)이 바뀌어 전체 miss. 도구 선택 결과에 따라 캐시 효과가 들쭉날쭉 — 별도 검토 필요.
  - **tool 루프 입력**: 한 spinx 호출 안의 tool 루프 회차마다 breakpoint 뒤 대화(tool_use/result)가 일반 입력으로 재과금된다(worker 호출당 평균 ~14.6K 미캐시). hexim tool 루프에서 회차마다 마지막 tool_result에 breakpoint를 거는 안 검토 중.
  - 역할별 판단을 정확히 하려면 spinx 호출 단위로 stage(plan/aggregate/worker/verifier)와 cache read/write를 남겨야 한다(현재 `agent_usage`는 run×model 단위).

## 로그 규약 (`spinx.go`, `orchestrator.go`)
- `spinx send request`(요청 전) 로그는 제거했다 — `spinx done`과 중복.
- 호출 1건이 끝나면 `spinx done`에 `spinx_id`, `input_bytes`, `tools`, `input_tokens`, `output_tokens`를 남긴다. 비용/입력 누적 추적의 핵심 로그.
  - 주의: hexim 캐시 분리 이후 `input_tokens`는 **캐시 제외** 값이다. 캐시 read/write는 이 로그에 없으므로 전체 입력·히트율은 `agent_usage`/spinx `spinx_message`의 캐시 컬럼으로 본다.
- `coordinator` 필드로 레이어를 구분한다(`stage`/`fanout`). 중첩되면 두 값이 다 붙는다(바깥→안쪽 순).
- `plan created` 로그로 Plan이 만든 계획(id/task/goal 요약)을 가시화한다 — empty goal 같은 문제를 바로 포착하는 데 썼다.
- 참고: `worker/runner/verifier/coordinator`의 start/done timer 로그는 `hexim.NewTimer`(공용 라이브러리) 소관이라 agent 쪽에서 끌 수 없다.

## 확인된 회귀 / 즉시 확인이 필요한 것
- ~~API invoke가 순차로 돈다~~ → invoke에서 Coordinator 분기를 제거하면서 이 회귀는 소멸했다(invoke는 이제 단일 Runner만). 병렬 관련 플래그는 `cmd/workflow`의 fanout Coordinator에만 존재한다(`Parallel: true` 하드코딩).
- **fanout Plan의 response_format 제거 효과는 Live로 재확인하지 못했다.** `planRequest`/`usesTools()` 변경 이후 workflow를 한 번도 돌리지 않았다. 다음 Live에서 fanout의 `plan created`에 goal이 모두 채워지는지, `orchestrator plan rejected`가 사라지는지 본다.
- **`cmd/workflow`의 Verifier는 선택 사항이 됐다.** `s.agent.Version.VerifierModel != ""`일 때만 `NewVerifier`를 만든다. 비어 있으면 Runner가 Verifier 없이(`nil`) 돌아 검증이 생략된다. 검증이 의도치 않게 빠지는지 버전 설정을 확인한다.
- `orchestratorModel` 상수는 현재 `anthropic/claude-sonnet-5-5`(stage/fanout Plan 공통). 값은 코드가 기준이다.

## 남은 과제
- 이중 Aggregate(fanout Aggregate + stage Aggregate) 중복 — 비싼 호출(input 40K~50K)이라 제거 검토.
- Verifier가 `pass:true`일 때 reason을 "placeholder"/빈 값으로 뱉는 경향(sonnet) — 판정은 맞으나 reason 부실. 특히 `pass:false`에 "placeholder"가 나오면 Worker가 고칠 방향을 못 받아 재시도가 맹목적이 된다. 미해결.
- 모델/파라미터 하드코딩(`orchestratorModel`, MaxToken/MaxSubTasks) — 버전 설정으로 받기.
- 오케스트레이션 전략(2계층 on/off, fanout 병렬 여부)을 **에이전트 버전 설정**으로 올리고, 조립을 `agent` 패키지의 공통 빌더로 추출해 `cmd/workflow`와 다른 진입점이 같은 조립 로직을 공유하는 방향. 현재는 2계층 조립이 `cmd/workflow/service.go:Run`에만 있어, 전파 시 중복이 생기지 않게 빌더로 모아야 한다. (단, fanout response_format 변경이 Live 미검증 상태라 그 검증 이후 착수 권장.)
- agent CLI(`cmd/agent`)는 단일 Runner 유지(upstream/2계층 미적용). invoke도 단일 Runner로 확정(위 비교표 참고).

## 작업 방식 메모
- 이 재설계는 Live 실행 로그를 보며 반복적으로 튜닝한 결과다. 변경 → Live 1회 실행 → 로그(`plan created`, `spinx done`, reject/재시도 패턴) 분석 → 원인 재확정 → 재수정의 순서를 따랐다. 가설이 로그로 반증되면 바로 정정했다(예: "haiku 역량 문제" → "Plan이 sonnet임을 확인 후 구조적 문제로 정정").
- 변경 커밋은 사용자가 git stage 단위로 직접 관리한다.

## 세션 운영 메모 (크레딧/토큰 절약)
- 세션이 길어지면 매 턴 전체 히스토리를 다시 처리하므로 턴당 비용이 커진다. 큰 주제가 끝나면 새 세션으로 옮기고, 이 스티어링으로 맥락을 복원한다.
- 가장 큰 토큰 소모는 **Live 로그 원문 붙여넣기**다. 스티어링·skill은 입력 토큰을 줄이지 못한다(오히려 로드된 텍스트만큼 입력이 늘어남). 입력 자체를 줄여야 한다.
  - 로그를 파일로 저장하고 경로만 전달 → 에이전트가 `execute_bash`로 요약 스크립트(jq/Python)를 돌려 출력만 본다. 원문은 컨텍스트에 안 들어온다.
  - 스크립트가 뽑을 항목: `spinx done`의 `input_tokens`/`output_tokens`를 `coordinator`·`runner_id`별 집계, `orchestrator plan rejected` 횟수와 사유, `worker did not pass`가 `max_attempts`까지 간 runner, `verifier verdict`의 reason이 `placeholder`/`x`/빈 값인 건.
  - LLM(MCP)으로 로그를 축약하는 방식은 컨텍스트는 줄여도 축약 LLM이 원문을 다시 입력받고, 단서가 한두 줄인 경우(빈 goal, 재시도 소진) 요약에서 빠질 수 있으며, 로그(workflow/run ID, 검색 결과)가 외부로 나가므로 비정형 부분에만 쓴다.
- 파일을 읽을 때는 필요한 범위만(offset/limit). `orchestrator.go`와 `prompt.go`는 프롬프트가 길어 통째로 읽으면 비싸다.
- 응답은 결론 위주로 짧게 요청하면 출력 토큰이 줄어든다.
- Power/skill은 아직 만들지 않았다. skill은 "절차의 일관성"을 위한 것이지 토큰 절감 수단이 아니다. 만든다면 후보는 (1) Live 로그 분석 절차, (2) "같은 모델이 반복 실패하면 구조 제약(도구+format 충돌 등)부터 의심" 추적 순서. `fileMatchPattern`의 중괄호 글롭(`{a,b}/**`) 지원 여부도 아직 확인하지 못했다. 로드가 안 되면 단일 글롭으로 바꾼다.
