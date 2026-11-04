---
inclusion: fileMatch
fileMatchPattern: 'hexim/**'
---

# HEXIM

여러 LLM 프로바이더(OpenAI, Anthropic, Google Gemini, AWS Bedrock)를 단일 인터페이스로 추상화한 Go 라이브러리.

- **모듈 경로**: `github.com/mz-team-ax/hexim`
- **Go 버전**: 1.26.3
- **경로**: `/Users/phddy/Projects/dx/hexim`
- **규모**: 비테스트 코드 약 7,900줄

## 패키지 구조

```
hexim/        # 루트: 범용 유틸리티 (제네릭 기반 slice, store, retry, cosine 등)
├── llm/      # 핵심: LLM 프로바이더 추상화 (가장 큰 패키지)
├── mcp/      # Model Context Protocol 서버/클라이언트
├── cache/    # Unix domain socket 기반 경량 캐시
├── errors/   # 스택/코드/필드를 담는 커스텀 에러 타입 (별칭 관례: er)
└── hextest/  # 테스트 헬퍼
```

## 핵심 설계 (llm 패키지)

### Client 인터페이스
프로바이더 통합의 핵심. `anthropic.go`, `openai.go`, `google.go`, `bedrock.go`가 각각 구현한다.

```go
type Client interface {
    Model() string
    tools(context.Context, *Request) (any, error) // unexported, 프로바이더별 tool 변환
    Invoke(context.Context, *Request) (*Response, error)
    InvokeStream(context.Context, *Request) (*Response, error)
}
```

`bedrock.go`가 가장 크며 Anthropic/Nova/Titan/TwelveLabs 다중 모델을 한 파일에서 처리한다.

### Request 빌더 (Options 패턴)
`NewRequest(WithMessage(...), WithTool(...), WithMaxToken(...), ...)` 형태로 구성.
- 기본값: MaxToken 3000, MaxToolCalls 3
- `ReasoningEffort`(none ~ xhigh) 지원

### 스트리밍 (채널 기반)
`Response`는 `event chan *Event`, `done chan error`, `quit chan struct{}`로 구성. `sync.Once`로 안전하게 close하며 `EventType`으로 텍스트/툴 호출/가드레일 등을 구분한다.

### Tool / Function Calling
- **Handler 방식**: Go 함수로 직접 실행
- **Hook 방식**: HTTP 콜백 (resty, URL placeholder 치환)
- `Choice`로 다중 결과 후처리/필터링
- Tool은 MD5 해시(`ToolHash`, canonical JSON 기반)로 식별 → MCP 툴 검색 캐싱에 활용
- MCP 서버 툴 등록은 별도 방식(`mcp.Tool` + `mcp.ToolHandler`) 사용

### Store 패턴 (전역 레지스트리)
제네릭 `hexim.Store[K,V]` 기반. 프로바이더별 모델 등록/조회 `clientStore`, `mcpStore`, `toolStore` 존재.

### 특징
- **Semantic Tool Search**: MCP 서버가 5개 초과면 코사인 유사도(`hexim.Cosine`)로 상위 3개 서버만 선택 후 후보 툴 10개로 추림
- **MCP 자동 헬스체크**: Ping 1분, Tools refresh 15분 주기, 재연결 로직
- **가드레일**: AWS Bedrock Guardrail 연동 (콘텐츠 필터/프롬프트 공격/민감정보)

## 코딩 관례
- 에러는 커스텀 `errors` 패키지 사용. import 별칭은 `er` (`er.New`, `er.Wrap`).
- 로깅은 `go.uber.org/zap` 사용.
- 제네릭 유틸리티는 루트 패키지에 위치.

## 알려진 개선 여지
1. `llm/request.go`의 `CandidateTools`에 디버그용 `println(...)` 잔존 → zap 로거로 교체/제거 권장
2. `mcp/client.go`의 `Connect`에서 `c.mu.TryLock()` 반환값을 무시하고 `Unlock()` 호출 → 경쟁 조건 위험
3. `bedrock.go`(약 40KB)에 모델별 로직이 집중 → 모델별 파일 분리 고려

## 테스트 / 빌드
- 테스트 실행: `cd /Users/phddy/Projects/dx/hexim && go test ./...`

## 리팩터링 방향 (진행 중 — llm 패키지 transport × model 분리)

> 상태: **설계 합의 전**. 아래는 코드 분석으로 확인된 문제와 가고자 하는 방향. 실제 구현은 미착수.

### 동기 (왜 바꾸려 하나)
현재 `llm`은 "프로바이더 중심" 설계라 **transport(어디로 보내나)와 model(capability/quirk)이 한 Client 구현에 뭉쳐** 있다. 두 가지가 아프다.
1. **같은 Anthropic 모델이 경로별로 통째 중복** — 직접 API(`anthropic.go`)와 Bedrock 경유(`bedrock.go`)가 메시지/툴/응답 변환과 tool_use 루프를 각자 재구현. SDK 타입만 다르고 로직은 사실상 2벌(`anthropic.TextBlockParam` vs `bedrocktyp.SystemContentBlockMemberText`).
2. **모델별 quirk가 로직에 하드코딩("땜빵")** — 예: Sonnet 4.5는 `tool_choice: any` 미지원, `thinking.type`을 `between_tools`/`disabled`로 분기, Nova는 "`<thinking>` 금지" 시스템 프롬프트 주입 등.

### 코드 분석으로 확인된 현재 구조 (근거 위치)
- `Client` 인터페이스(`llm/client.go:8-13`): `Model() / tools(...)(unexported) / Invoke / InvokeStream`. `tools`가 unexported라 **외부 패키지에서 Client 구현 불가** → 프로바이더 추가는 `llm` 내부에서만.
- Bedrock은 이미 반쯤 분리됨: 공용 `*bedrock`(bedrockruntime client, transport)을 `Bedrock`/`BedrockEmbedding`/`BedrockRerank`/`BedrockTwelvelabs`/`BedrockGuardrail`가 임베드. 생성자는 서로 다른 타입 반환.
- quirk/중복이 몰린 지점:
  - `bedrock.go` `payload`/`payloadStream` 두 곳에 **같은 분기 복붙** — `strings.Contains(model, "anthropic.claude-sonnet-5-5")` → `thinking=between_tools`, else `disabled`.
  - `splitMessages`의 `strings.Index(model, "nova")` → Nova 전용 프롬프트.
  - reasoning/thinking 매핑이 4개 프로바이더에 전부 다른 방식(구조체 vs map)으로 흩어짐.
  - "마지막 툴 콜 턴엔 툴 끄기"(`i == MaxToolCalls-1`)가 각 파일에 분산(Anthropic `Tools=nil`, Google `Tools=nil`, OpenAI/Bedrock 프롬프트만).
  - cache(`WithCache`) 적용 지점이 프로바이더별로 분산(Anthropic `CacheControl`, Bedrock `CachePoint`, OpenAI/Google 미적용).
  - 미지원 기능은 런타임 에러로 표현(`BedrockEmbedding.InvokeStream` "does not support streaming" 등).
- `Store`(`llm/store.go`): `ProviderType`(`openai/anthropic/google/bedrock`) → `client.Model()` 2단계 맵. **Bedrock의 anthropic/nova/titan/twelvelabs가 전부 단일 `ProviderBedrock` 아래 평평하게** 들어가, model capability 구분은 model 문자열에만 의존.
- `NewResponse`의 provider 문자열이 이미 `"bedrock-anthropic"`처럼 **transport+model을 합쳐** 넣음 → 분리 시 자연스러운 경계.

### 목표 구조 (추천안: Transport × ModelSpec)
- `Transport` — "바이트를 어디로 보내나"만 책임 (`AnthropicDirect` / `BedrockRuntime` / `OpenAI` / `GoogleGenAI`). SDK 클라이언트 보유.
- `ModelSpec` — 모델별 capability/quirk를 **데이터(플래그)로 선언**. 예: `ToolChoiceAny bool`, `Thinking ThinkingMode`, `SupportsPDF/Cache bool`, `DefaultMaxToken int64`. quirk를 `if strings.Contains(...)`가 아니라 `spec.ToolChoiceAny` 조회로.
- 같은 Anthropic 메시지/툴 변환을 직접·Bedrock 두 transport가 공유. 모델명 → spec **레지스트리**로 명시 등록(미등록은 보수적 기본 spec fallback).
- 대안 B(침습 최소): 구조 유지하고 흩어진 `strings.Contains(model,...)`만 중앙 `modelQuirks` 맵으로 추출. 대안 C(완전 재설계: Codec+Transport+Spec 3분리)는 범위 과대.

### 미확정 결정 사항 (다음에 이어서 정할 것)
1. **공개 API 호환성**: `NewAnthropic(model,key)`/`NewBedrock(model,cfg)` 시그니처 유지하고 내부만 교체할지, `NewClient(spec, transport)` 신 API + 기존 deprecated로 갈지. (소비 레포 4개 영향)
2. **모델 식별**: `ModelSpec` 명시 레지스트리(sonnet-4.5, nova-pro 등 테이블) 도입 여부 + 미등록 fallback 정책.
3. **범위**: Anthropic(직접/Bedrock) 이원화 + quirk만 먼저 잡을지, OpenAI/Google/Titan/TwelveLabs까지 한 번에 바꿀지.

### 착수 전 유의
- 공개 API(`Store().Set/Get`, `NewRequest`, `Invoke/InvokeStream`)는 가능한 유지해 소비 레포(`aidea-api-agent/workflow/spinx`) 영향 최소화.
- 리팩터링 전후로 `go test ./...`(특히 `*_test.go`, `_Live` 제외)로 회귀 확인.
