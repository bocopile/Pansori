# SPEC: modules/contracts/ — Inter-module Contracts (proto + openapi)

| 메타 | 값 |
|---|---|
| **상태** | Draft |
| **작성일** | 2026-05-18 |
| **오너** | TBD |
| **관련 ADR** | ADR-0003 (라이선스 정책), ADR-0003-1 (Apache-2.0) |
| **영향 모듈** | (전체 — `contracts/`는 모든 모듈이 의존) |
| **구현 시점** | **P1 진입 시 SubOrchestrator가 본 SPEC 기반 dispatch** |

---

## 1. 목적

`modules/contracts/`는 본 프로젝트의 **인터페이스 단일 출처(SSOT)**다. CLAUDE.md §8 문서 우선순위 3순위로 ADR/SPEC 다음이다.

규칙:
- 모든 모듈 간 통신(에이전트↔서버, 서버↔UI, operator CRD)은 `contracts/`에 정의된 스키마만 사용
- 모듈 간 Go struct 직접 참조 금지 — `contracts/`에서 생성된 타입만 import
- `contracts/` 변경 정책: non-breaking은 자유, breaking은 ADR 필수 (상세 §6)

---

## 2. 디렉터리 구조

```
modules/contracts/
├── README.md                          # 본 SPEC 요약 + 빌드 명령
├── buf.yaml                           # buf v2 workspace config (https://buf.build/docs/configuration/v2/buf-yaml/)
├── buf.gen.yaml                       # 코드 생성 설정 (v2)
├── proto/                             # gRPC / OTLP gRPC 메시지 정의
│   └── coroot/
│       └── v1/
│           ├── agent.proto            # 에이전트↔서버 control plane
│           ├── telemetry.proto        # 자체 텔레메트리 메시지 (OTLP 보조)
│           ├── topology.proto         # service map 데이터 모델
│           ├── inspection.proto       # SLO/이상탐지 결과 모델
│           └── errors.proto           # 공통 에러 모델 (RFC 7807 + 자체 코드)
├── openapi/                           # REST API 정의
│   ├── api.yaml                       # 메인 API (UI ↔ server)
│   ├── webhooks.yaml                  # 인바운드 webhook (사내 알람 게이트웨이 → server)
│   └── common.yaml                    # 공통 components (에러 모델, pagination 등)
├── gen/                               # 코드 생성 산출물 (커밋 X, .gitignore)
│   ├── go/                            # buf 생성 Go 산출 (coroot/contracts/gen/go/v1)
│   │   ├── v1/                        # proto → Go (protoc-gen-go + protoc-gen-go-grpc)
│   │   └── api/                       # OpenAPI → Go (oapi-codegen, types + server stub)
│   └── ts/                            # UI용 산출
│       ├── v1/                        # proto → TS (@bufbuild/protoc-gen-es)
│       └── api/                       # OpenAPI → TS (openapi-typescript)
└── examples/                          # 각 메시지 샘플 JSON/YAML (테스트용)
```

> **gen/ 위치 결정**: 단일 위치 `modules/contracts/gen/` (모든 모듈이 이곳을 import). 모듈별 분산은 import path 중복으로 인한 코드 생성 drift 위험.
>
> **CRD 표현 결정**: kubebuilder Go struct 직접 정의 (operator 모듈 내). proto 기반 CRD는 운영자 도구 호환성 손실 — 채택 안 함. operator 모듈은 `contracts/`를 import해 통신용 타입만 공유.

### 2.1 명명 규약
- **proto package**: `coroot.v1` (사내 prefix 없음 — 사내 조직명 확정 시 별도 ADR로 마이그레이션)
- **proto file**: `<domain>.proto` (snake_case, 단수형)
- **proto message**: `PascalCase` (`ServiceNode`, not `ServiceNodes`)
- **proto service**: `<Domain>Service` (`AgentRegistryService`)
- **proto rpc**: `<Verb><Noun>` (`RegisterAgent`, `StreamTelemetry`)
- **proto field**: `snake_case`, semantic name (`pod_name`, not `pn`)
- **proto file `go_package` 옵션**: 필수 — `coroot/modules/contracts/gen/go/v1;corootv1`
- **proto field number**:
  - 1~15: 핵심 hot path 필드 (1-byte tag)
  - 16~2047: 일반 필드
  - 19000~19999: **예약** (proto 내부용, 사용 금지)
  - **삭제된 필드 번호는 `reserved` 선언 필수** — 재사용 절대 금지
- **OpenAPI path**: **kebab-case** (`/v1/service-map`, `/v1/inspections`) — 사내 표준 미정 시 본 결정 적용
- **OpenAPI operationId**: `<verb><Resource>` (camelCase: `listServices`, `getServiceMap`)

### 2.2 버전 관리
- **proto**: package에 `v1`, `v2` 같은 major suffix. **breaking change는 새 major** (`v2` 디렉터리 신설). minor는 ADR 없이 추가 가능
- **OpenAPI**: `info.version: 1.x.x` semver. breaking change = major bump + 별도 `openapi/v2/api.yaml`
- 모든 breaking change = ADR 필수 (`docs/30-adr/NNNN-contracts-vN.md`)

---

## 3. 빌드 도구 (확정)

| 도구 | 용도 | 라이선스 |
|---|---|---|
| `buf` (`bufbuild/buf`) | proto lint, format, breaking change 검출, codegen | Apache-2.0 ✅ |
| `protoc-gen-go` + `protoc-gen-go-grpc` | proto → Go | BSD-3-Clause ✅ |
| `@bufbuild/protoc-gen-es` | proto → TS (Connect-RPC 호환) | Apache-2.0 ✅ |
| `oapi-codegen` (`oapi-codegen/oapi-codegen`) | OpenAPI → Go (server + types) | Apache-2.0 ✅ |
| `openapi-typescript` | OpenAPI → TS types | MIT ✅ |
| `spectral` (`stoplightio/spectral`) | OpenAPI lint | Apache-2.0 ✅ |

모두 Apache-2.0과 호환 (ADR-0003-1 §2.3).

### 3.0 프로토콜 선택 매트릭스

| 통신 페어 | 프로토콜 | 사유 |
|---|---|---|
| 에이전트 ↔ 서버 (텔레메트리) | OTLP gRPC | 표준 + bandwidth 효율 |
| 에이전트 ↔ 서버 (control plane) | gRPC (`coroot.v1.AgentService`) | bidirectional streaming 필요 (heartbeat) |
| UI ↔ 서버 | REST/JSON (OpenAPI) | 브라우저 친화, debugging 용이 |
| Operator ↔ K8s API | client-go (gRPC underlying) | 표준 |
| 인바운드 webhook (사내 알람 등) | REST/JSON (`webhooks.yaml`) | 외부 호환성 |

### 3.1 빌드 명령 (SubOrchestrator 구현 후)
```bash
# proto 빌드 (lint + breaking + gen) — gen/ 위치는 §2 디렉터리 구조 참조
buf lint
buf breaking --against '.git#branch=main'
buf generate                                                       # → modules/contracts/gen/go/v1/, gen/ts/v1/

# OpenAPI 빌드
spectral lint modules/contracts/openapi/api.yaml
oapi-codegen -config modules/contracts/openapi/oapi-codegen.yaml \
  modules/contracts/openapi/api.yaml                                # → modules/contracts/gen/go/api/api.gen.go
npx openapi-typescript modules/contracts/openapi/api.yaml \
  -o modules/contracts/gen/ts/api/api.ts                            # → modules/contracts/gen/ts/api/api.ts
```

> ⚠️ 본 SPEC의 codegen 출력 경로(`modules/contracts/gen/...`)가 SSOT. 다른 문서(CLAUDE.md §6 예시 등)에 다른 경로가 보이면 본 SPEC을 따른다.

### 3.2 재현성 강제 (CI)
- 모든 PR에서 `buf generate` + OpenAPI codegen을 실행
- 결과 산출물(`modules/contracts/gen/`)은 `.gitignore`로 커밋 제외
- **단**: CI에서 `go build ./...`를 실행하기 전에 codegen 단계가 반드시 선행. 검증 명령(CLAUDE.md §6) 진입점에 `make contracts` 또는 `go generate ./...` 호출 추가 (`Makefile`은 P1에서 생성 — `go generate ./modules/contracts/...` 단일 진입점 권장)

---

## 4. P1 진입 시 최우선 정의할 인터페이스 (Top-10)

이 10개가 `modules/contracts/`의 P1 v0.1 산출물. 우선순위 순.

| # | 인터페이스 | 파일 | 책임 | 사용 모듈 |
|---|---|---|---|---|
| 1 | `Agent.Register` | `proto/agent.proto` | 에이전트 등록 + 인증서 발급 | agent-node/cluster, server |
| 2 | `Agent.Heartbeat` | `proto/agent.proto` | 에이전트 생존 신호 | agent-*, server |
| 3 | OTLP 수신 (Trace/Metric/Log) | (외부 OTel spec 직접 인용) | 표준 OTLP 수신 — 본 프로젝트가 정의하는 게 아니라 OTel 표준을 그대로 따름 | transport, agent-* |
| 4 | `Topology.ServiceNode` / `Edge` | `proto/topology.proto` | service map 그래프 노드/엣지 | processing, api, ui |
| 5 | `Inspection.Finding` | `proto/inspection.proto` | SLO 위반/이상 탐지 결과 | processing, api, ui, alarm |
| 6 | `Telemetry.Sample` (보조) | `proto/telemetry.proto` | OTLP 외 자체 보조 메시지 (예: 에이전트 자체 진단) | agent-*, server |
| 7 | REST `/v1/services` | `openapi/api.yaml` | 서비스 목록 조회 | ui, api |
| 8 | REST `/v1/services/{id}/topology` | `openapi/api.yaml` | service map 조회 | ui, api |
| 9 | REST `/v1/services/{id}/inspections` | `openapi/api.yaml` | inspection 결과 조회 | ui, api |
| 10 | REST `/v1/auth/session` | `openapi/api.yaml` | 세션 정보 (SSO 통합) | ui, api, auth |
| 11 | `webhooks.yaml` 인바운드 webhook | `openapi/webhooks.yaml` | 사내 알람 게이트웨이 등 외부 진입점 | api, processing |

> P1 v0.1 머지 조건: 위 11개 + `buf generate` + `oapi-codegen` 통과 + 각 메시지의 `examples/` 샘플 1개씩.

### 4.1 공통 컴포넌트 (`openapi/common.yaml`)

다음은 모든 REST endpoint가 공유:

#### 4.1.1 에러 모델 (RFC 7807 Problem Details 확장)
```yaml
components:
  schemas:
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type: { type: string, format: uri }       # 에러 분류 URI
        title: { type: string }                    # 사람용 짧은 설명
        status: { type: integer }                  # HTTP status
        detail: { type: string }                   # 사람용 상세
        instance: { type: string, format: uri }    # 이번 요청의 식별자
        code: { type: string }                     # 사내 에러코드 (예: COROOT-AUTH-001)
        trace_id: { type: string }                 # 분산 추적 ID
```
- HTTP 4xx/5xx 응답은 모두 `Content-Type: application/problem+json`
- `code` 네임스페이스: `COROOT-<MODULE>-<NNN>` (예: `COROOT-API-401`, `COROOT-PROC-503`)

#### 4.1.2 Pagination (Cursor 기반)
```yaml
parameters:
  pageSize:    { name: page_size, in: query, schema: { type: integer, default: 50, maximum: 500 } }
  pageCursor:  { name: cursor,    in: query, schema: { type: string } }
responses:
  PageMeta:
    description: 응답 헤더로 페이지네이션 메타
    headers:
      X-Next-Cursor: { schema: { type: string } }
      X-Total-Count: { schema: { type: integer } }      # 옵션 (대량 시 omit 가능)
```

#### 4.1.3 인증 / Security Scheme
```yaml
components:
  securitySchemes:
    sessionCookie:
      type: apiKey
      in: cookie
      name: coroot_session
    serviceBearer:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - sessionCookie: []
```
- UI ↔ server: `sessionCookie` (HttpOnly, Secure, SameSite=Strict)
- 사외 webhook 인바운드: `serviceBearer` (사내 발급 JWT) 또는 mTLS — `webhooks.yaml`에서 명시

#### 4.1.4 표준 헤더
| 헤더 | 방향 | 의미 |
|---|---|---|
| `X-Request-Id` | 양방향 | 요청 추적 (없으면 server가 생성하고 응답에 첨부) |
| `X-Trace-Id` | 응답 | OTel trace ID — 디버깅용 |
| `X-RateLimit-Remaining` | 응답 | 남은 quota (해당 시) |

---

## 5. 의존성 규칙

```
ui ──────────────────► api (REST)
                       │
agent-* ──────────────► transport (OTLP/PromRW) ──► storage
                       │
processing ──► storage ──► (Prom/CH/PG)
processing ──► [외부] 사내 Alarm Gateway (HTTPS, contracts/openapi/webhooks.yaml)
operator ───► CRDs (kubebuilder Go struct + contracts/proto/v1 import)

전체 의존성 그래프에서 contracts/는 "data plane"이 아니라 "type SSOT"
→ 모든 모듈이 contracts/gen/go/* 만 import
→ 모듈 간 직접 type 참조는 금지
```

> **alarm 명확화** (★ 방향 단일화):
> - `webhooks.yaml`은 **인바운드 webhook 전용** — 외부 시스템이 본 서버를 호출하는 경우 (예: 사내 알람 게이트웨이로부터 ack/retry, GitOps 시스템의 sync 알림 등)
> - **아웃바운드** — `processing` 모듈이 사내 Alarm Gateway로 보내는 HTTPS 호출 — 은 `webhooks.yaml`이 아니라 **사내 게이트웨이가 제공하는 OpenAPI spec을 client로 import**해서 처리 (vendor side)
> - `modules/alarm/` 자체는 존재하지 않음 (별도 모듈 아님)

### 5.1 import 규칙
- ✅ `import "coroot/modules/contracts/gen/go/v1"`
- ❌ `import "coroot/modules/agent-node/internal/types"` (모듈 간 internal 참조)
- ❌ `import "github.com/coroot/coroot/..."` (Coroot 원본 import — ADR-0003)

---

## 6. Breaking Change 정책

### 6.1 Non-breaking (자유)
- 신규 field 추가 (proto3 기본값 사용)
- 신규 RPC / endpoint 추가
- 신규 enum value 추가 (단, 클라이언트가 unknown을 graceful 처리해야)
- 주석/문서 변경

### 6.2 Breaking (ADR 필수 + 새 major)
- 기존 field 제거 / 타입 변경 / number 변경
- 기존 RPC / endpoint 제거 또는 시그니처 변경
- enum value 제거
- `required` 의미 강화 (proto3는 모두 optional이나 운영상 필수 약속이 깨지는 경우)

### 6.3 Deprecation
- 즉시 제거 금지. `[deprecated = true]` 마킹 후 최소 1 minor 유지
- 제거 시 ADR + CHANGELOG 명시

### 6.4 CI 강제
- `buf breaking --against '.git#branch=main'` 통과 필수
- 위반 시 PR 빌드 실패 (ADR 첨부 시 manual override)

---

## 7. Apache-2.0 SPDX 헤더

모든 `.proto`, `.yaml`, `.go` (생성 제외) 파일 상단:

```proto
// SPDX-FileCopyrightText: 2026 <사내 조직명 — TBD>
// SPDX-License-Identifier: Apache-2.0

syntax = "proto3";
package coroot.v1;
```

```yaml
# SPDX-FileCopyrightText: 2026 <사내 조직명 — TBD>
# SPDX-License-Identifier: Apache-2.0
openapi: 3.1.0
info:
  title: Coroot API
  version: 0.1.0
```

생성된 코드(`gen/`)는 `.gitignore`로 커밋 제외 — 빌드 시 재생성.

---

## 8. 테스트 / 검증

| 검증 | 도구 | 시점 |
|---|---|---|
| proto syntax + lint | `buf lint` | PR / pre-commit |
| proto breaking change | `buf breaking` | PR |
| OpenAPI lint | `spectral lint` | PR |
| 의존성 라이선스 (Apache-2.0 호환) | `go-licenses report ./...` | PR |
| `examples/` 샘플의 스키마 유효성 | 각 메시지에 대한 unmarshal 테스트 | CI |

---

## 9. 결정 완료 사항 (이전 Open Questions)

- [x] `gen/` 위치: **`modules/contracts/gen/`** 단일 위치 (§2 결정)
- [x] OpenAPI path 컨벤션: **kebab-case** (§2.1 결정)
- [x] proto 패키지 prefix: **`coroot.v1`** — 사내 prefix 없이 시작 (§2.1). 사내 조직명 확정 시 별도 ADR로 마이그레이션
- [x] CRD: **kubebuilder Go struct** 직접 정의 (§2 결정)
- [x] `webhooks.yaml` 인증: **mTLS 또는 사내 JWT Bearer** — `webhooks.yaml`에서 endpoint별 명시 (§4.1.3)
- [x] gRPC 프로토콜 옵션: **`@bufbuild/protoc-gen-es`로 Connect-RPC 호환 코드 생성** — gRPC-Web 옵션 포함 (§3 도구 표)

## 9.1 남은 Open Questions (P1 진행 중 결정)

- [ ] `Makefile` vs `go generate` 단일 진입점 — P1 첫 PR에서 결정
- [ ] sub-language(Go 1.21+ workspace) 사용 여부 — `modules/contracts/`만 별도 module로 둘지
- [ ] gRPC reflection 활성화 정책 (디버깅 vs 보안)
- [ ] 사내 조직명 / copyright holder 확정 (ADR-0003-1 §3.3 참조)

---

## 10. 후속 산출물 (P1에서 SubOrchestrator가 생성)

순서:
1. `modules/contracts/` 디렉터리 + `README.md` + `buf.yaml` + `buf.gen.yaml`
2. `proto/coroot/v1/agent.proto` (Register + Heartbeat — Top-10 #1, #2)
3. `proto/coroot/v1/topology.proto` (#4)
4. `proto/coroot/v1/inspection.proto` (#5)
5. `proto/coroot/v1/telemetry.proto` (#6)
6. `openapi/api.yaml` (#7~#10)
7. `examples/` 샘플 + 단위 테스트
8. CI 통합 (`buf lint/breaking/generate`, `spectral lint`, `oapi-codegen`)

각 단계는 SubOrchestrator가 본 SPEC을 입력으로 dispatch.
