# CLAUDE.md — 프로젝트 루트 컨텍스트 (Slim)

> **이 파일의 위치**: 본 프로젝트에서 일하는 모든 AI 에이전트(Claude / Codex / Gemini CLI / SubOrchestrator 하위 에이전트)가 **가장 먼저** 읽는 파일.
> **이 파일의 약속**: 약 220줄 이하 유지 (±10줄). 상세는 분리 문서 링크. 검증 명령 SSOT는 §6.
> **용어**: **MUST/MUST NOT** = 위반 시 PR 거부 + AI 즉시 거부 · **SHOULD** = 합당한 사유 없이 위반 금지 (ADR로 예외 기록) · **MAY** = 자유.

---

## 1. ⚡ Agent Quickstart (작업 유형 라우터)

```
0) 현재 단계 = P0 (스펙·인터페이스 정의)
   → modules/ 등 비즈니스 디렉터리 아직 미생성 (§5 트리 마커 참조)
   → 코드 작성 전 contracts/spec/ADR 합의 우선

1) MUST NOT (즉시 거부 — 작업 유형 무관 공통):
   - Coroot(또는 어떤 AGPL) 코드를 본 저장소에 가져오지 말 것 (clone/copy/번역/refactor 모두 금지 — ADR-0003 순수 영감)
   - Coroot 코드를 보면서 우리 코드를 작성하지 말 것 (공식 문서·논문·표준만 참조)
   - 외부 SaaS / 클라우드 SDK 직접 호출
   - 로컬 인증 추가 (사내 SSO만)
   - K8s v1.34 미만 API 사용

2) 작업 유형 선택 → 해당 행만 로딩:
   ┌─────────────────────┬─────────────────────────────────────────────────────────┐
   │ 작업 유형           │ 먼저 볼 문서 (이것만 읽으면 충분)                        │
   ├─────────────────────┼─────────────────────────────────────────────────────────┤
   │ ADR/SPEC 신규/수정  │ docs/30-adr/, docs/20-specs/, 본 파일 §3·§4              │
   │ 인터페이스 변경     │ modules/contracts/ + 영향 모듈 SPEC (P0 단계는 합의안만) │
   │ 새 모듈 구현(P1+)   │ 본 §5 진입점 표 → 해당 spec → contracts                  │
   │ 버그 수정           │ 실패 테스트 작성 → 재현 → 패치 (해당 모듈 spec만 참조)   │
   │ K8s API 사용        │ docs/30-adr/0002-k8s-feature-state-table.md (canonical)  │
   │ 보안/배포 표준      │ docs/20-specs/onprem-baseline-spec.md                    │
   │ 문서·번역 초안      │ docs/01-architecture.md + glossary.md                    │
   └─────────────────────┴─────────────────────────────────────────────────────────┘

3) 작업 후 SSOT 검증 (§6 명령 복사 실행):
   go build ./... && go test ./... && go vet ./... && golangci-lint run && govulncheck ./...

4) 사용자에게 먼저 질문해야 하는 작업:
   새 외부 의존성 / public API 변경 / auth·RBAC 변경 / K8s API surface 변경
```

---

## 2. 정체성

**한 문장**: Coroot(eBPF 옵저버빌리티) 벤치마크 → On-Premise(사내망+제한 외부) K8s v1.34+ 환경의 한국형 옵저버빌리티 플랫폼.

- **스택**: Go 1.26 (백엔드/에이전트) + Vue 3 (UI) + ClickHouse + Prometheus + PostgreSQL
- **라이선스 정책**: **순수 영감 / Pure Inspiration (ADR-0003)** — Coroot 코드 어떤 형태로든 본 저장소에 두지 않음. 공식 문서·논문·표준에서만 학습. 법무 검토 불필요
- **본 프로젝트 SPDX**: **Apache-2.0** (ADR-0003-1) — 모든 신규 파일 상단에 `SPDX-License-Identifier: Apache-2.0` 헤더 필수
- **배포 환경**: K8s v1.34+ (Native), VM/베어메탈 (옵션), Air-gapped

---

## 3. HARD CONSTRAINTS

### 3.1 외부 호출 (MUST NOT)
- ❌ 외부 SaaS 직접 호출 (Datadog, NewRelic, OpenAI/Anthropic API, AWS/GCP/Azure SDK)
- ❌ 하드코딩된 외부 도메인 (`api.openai.com`, `*.amazonaws.com` 등)
- ❌ proxy 환경변수 무시하는 HTTP 클라이언트
- ✅ 모든 외부 호출은 `modules/transport/` 또는 사내 게이트웨이 경유

### 3.2 인증 / 시크릿 (MUST)
- ✅ 사내 SSO (OIDC/SAML) 강제. 로컬 username/password 처리 금지
- ✅ 모든 에이전트↔서버 통신은 **mTLS** (사내 PKI)
- ❌ 시크릿/토큰 코드 내 하드코딩. 환경변수 또는 사내 Vault만

### 3.3 K8s (MUST)
- ✅ minimum **v1.34** / primary **v1.36** (ADR-0001)
- ✅ `k8s.io/client-go v0.36.x` + `controller-runtime v0.24.x` 고정
- ✅ 신규 K8s 기능은 **API discovery로 capability gating** (T1/T2/T3 분류는 `coroot-mapping.md`)
- ❌ deprecated/removed API 사용 금지 (`PodSecurityPolicy`, `batch/v1beta1`, `Endpoints` 직접 watch 등 — 상세 `ADR-0002`)

### 3.4 표준 준수 (MUST)
- ✅ 모든 로그·메트릭은 OpenTelemetry 표준
- ✅ 모든 API는 OpenAPI 3.1 (`modules/contracts/openapi/`)
- ✅ 모든 PII는 마스킹 또는 사내 정책 준수
- ✅ 외부 호출에 timeout + retry + circuit breaker

---

## 4. 안티패턴 (AUTO-REJECT / WARN)

| 안티패턴 | 강도 | 사유 / 올바른 방법 |
|---|---|---|
| `http.Get("https://...")` 직접 호출 | **AUTO-REJECT** | proxy/timeout 누락 → `pkg/httpx.Client` (⏳ P0 합의안) |
| `os.Getenv` 코드 곳곳 | **WARN** | 설정 진입점 단일화 → `internal/config` (⏳ P0 합의안) |
| ClickHouse SQL 문자열 결합 | **AUTO-REJECT** | SQL injection → 파라미터 바인딩 |
| 라이브러리 코드에서 `panic` | **AUTO-REJECT** | 서비스 다운 → `error` 리턴 |
| 로그에 토큰/PII 출력 | **AUTO-REJECT** | `pkg/log/redact` 마스킹 (⏳ P0 합의안) |
| Coroot(또는 AGPL) 코드를 본 저장소에 도입 | **AUTO-REJECT** | ADR-0003 순수 영감 — 공식 문서·논문에서 학습 후 자체 구현 |
| Coroot 코드를 열어두고 우리 코드 작성 | **AUTO-REJECT** | ADR-0003 §2.3 — 시간·공간적 분리 필요. 막히면 사용자에게 보고 |
| `interface{}` / `any` 남발 | **WARN** | 제네릭/구체 타입 |
| `corev1.Endpoints` watch | **AUTO-REJECT** | v1.33 deprecated → `discoveryv1.EndpointSlice` |
| `policy/v1beta1.PodSecurityPolicy` | **AUTO-REJECT** | v1.25 제거 → PSA |
| init container + `sleep infinity` sidecar | **AUTO-REJECT** | Native Sidecar (`restartPolicy: Always`) |
| `bcc` in-pod 컴파일 | **AUTO-REJECT** | `cilium/ebpf` (CO-RE) v0.21.x |
| `cluster-admin` 부여 | **AUTO-REJECT** | 최소권한 (`k8s-integration-spec.md` §6) |
| `aws-sdk-go-v2` / `cloud.google.com/go` import | **AUTO-REJECT** | On-Prem 무관 |
| AI 워크플로: 모듈 CLAUDE.md 미확인 | **WARN** | 작업 시작 전 §1 Quickstart 따르기 |
| AI 워크플로: 검증 명령 결과 미보고 | **WARN** | §1.5 검증 명령 실행 + 결과 보고 |
| AI 워크플로: generated 파일 직접 수정 | **AUTO-REJECT** | `*_gen.go`, `*.pb.go`는 생성기로만 |

---

## 5. 디렉터리 + 작업 진입점

> ⚠️ **현재(P0) 워크스페이스 상태**: P0는 **문서 정의 단계**. 비즈니스/가드레일 디렉터리는 모두 **합의안 — 미생성**. 실제 파일·코드 생성은 SubOrchestrator가 P1 시작 시 ADR/SPEC 사양 기반으로 dispatch.

```
coroot/
├── CLAUDE.md                  # ✅ 존재 (본 파일)
├── docs/                      # ✅ 존재 — 문서 (상세는 §9)
├── go.mod, main.go            # ✅ 존재 — Go 모듈 + CLI 진입점
├── .claude/                   # ✅ 존재 — agent-roles.md 등
├── modules/                   # ⏳ P0: 합의안 — 비즈니스 코드
│   ├── contracts/             # proto/openapi (모든 모듈이 의존)
│   ├── agent-node/            # eBPF 노드 에이전트
│   ├── agent-cluster/         # K8s/VM 클러스터 수집기
│   ├── agent-db-postgres/, agent-db-mysql/
│   ├── transport/             # OTLP, PromRW
│   ├── storage/               # metrics, telemetry, meta
│   ├── processing/            # topology, inspections, rca, logparser
│   ├── api/                   # REST API
│   ├── auth/                  # SSO/RBAC
│   ├── operator/              # K8s Operator Go 코드
│   └── ui/                    # Vue 3 SPA
├── pkg/                       # ⏳ P0: 합의안 — 공유 유틸 (tls/httpx/log/errs)
├── deploy/                    # ⏳ P0: 합의안 — helm/operator/systemd/docker 매니페스트
├── experiments/               # ⏳ P0: 합의안 — 일반 R&D 영역 (AGPL 코드 금지, ADR-0003 §2.4)
├── .github/, scripts/         # ⏳ P0: 합의안 — PR template + 간이 의존성 license 체크 (SubOrchestrator가 P1에서 생성)
└── tests/                     # ⏳ P0: 합의안

# ❌ reference/ 디렉터리는 ADR-0003에 따라 생성하지 않음 (Coroot 코드 clone 금지)
```

### 작업 유형별 진입점
| 작업 유형 | 먼저 볼 파일 | 다음 단계 |
|---|---|---|
| 새 기능 구현 | `docs/20-specs/<name>-spec.md` | `modules/contracts/` → `modules/<name>/` |
| 버그 수정 | 실패 테스트 작성 → 재현 | `modules/<name>/`만 수정 |
| 인터페이스 변경 | `modules/contracts/` | 영향 모듈 전체 갱신 + SPEC 갱신 |
| 의존성 추가 | `docs/contributing.md §1` | ADR 필수 |
| 아키텍처 결정 | `docs/30-adr/` 신규 ADR | 영향 SPEC/문서 동시 갱신 |
| K8s API 사용 | `ADR-0002` 상태표 확인 | `k8s-integration-spec.md` |
| 보안/배포 표준 | `onprem-baseline-spec.md` | PSA/NetworkPolicy 동봉 |

---

## 6. 핵심 명령어

```bash
# 빌드
go build ./...

# 검증 원샷 (codegen 선행 — contracts-spec §3.2 요구)
go generate ./modules/contracts/... && go build ./... && go test ./... && go vet ./... && golangci-lint run && govulncheck ./...

# 단일 패키지
go test ./modules/<name>/...
go test -run <TestName> ./modules/<name>/...

# 탐색 (코드 위치)
rg "<keyword>" modules/
rg --files modules/<name>/

# UI
cd modules/ui && npm run dev          # 개발
cd modules/ui && npm run typecheck    # 타입 체크
cd modules/ui && npm run build        # 빌드
cd modules/ui && npm run test         # 테스트

# 코드 생성
buf generate
oapi-codegen -config modules/contracts/openapi/oapi-codegen.yaml modules/contracts/openapi/api.yaml   # → modules/contracts/gen/go/api/api.gen.go (상세는 contracts-spec.md §3)
```

---

## 7. 위임 가이드 (요약)

| 작업 | 1차 담당 | 비고 |
|---|---|---|
| 사실 검증·라이브러리 버전 | **Codex** | 웹 검색 + 공식 출처 |
| 아키텍처·리뷰·SSOT 정합성 | **Claude** | 라인 정밀 |
| 문서/요약/번역 초안 | **Gemini** | 단독 사실 검증 금지 |
| 작은 Go 구현/테스트 | **Codex** | 즉시 patch 우선 |
| 교차 검증 | **3사 병렬** | 최종 합의는 사용자 |

> 상세: `.claude/agent-roles.md` (SubOrchestrator dispatch 룰)

---

## 8. 문서 우선순위 (충돌 시 — 7단계)

```
1. ADR (docs/30-adr/*)
2. SPEC (docs/20-specs/*)
3. Contracts (modules/contracts/ — proto/openapi)  ← 인터페이스 SSOT
4. 아키텍처 (docs/01-architecture.md)
5. 매핑표 (docs/10-benchmarks/coroot-mapping.md)
6. 루트 CLAUDE.md (본 파일)
7. 모듈 CLAUDE.md (modules/<name>/CLAUDE.md)
```
모듈 CLAUDE.md가 상위 문서를 위반하면 무효 → 사용자에게 즉시 보고.

---

## 9. 빠른 참조 링크

| 문서 | 용도 |
|---|---|
| `docs/01-architecture.md` | 시스템 그림 SSOT |
| `docs/10-benchmarks/coroot-mapping.md` | Coroot ↔ 우리 모듈 + Tier 분류 |
| `docs/20-specs/k8s-integration-spec.md` | K8s API 활용 상세 |
| `docs/20-specs/onprem-baseline-spec.md` | PSA/PKI/eBPF/카디널리티/Air-gapped |
| `docs/20-specs/contracts-spec.md` | **modules/contracts/ 인터페이스 사양** (P1 진입점) |
| `docs/30-adr/0001-k8s-version-policy.md` | K8s 버전 정책 |
| `docs/30-adr/0002-k8s-feature-state-table.md` | K8s feature canonical 상태표 |
| `docs/30-adr/0003-license-policy.md` | **라이선스 정책 — 순수 영감** (★ 필독) |
| `docs/30-adr/0003-1-license-spdx.md` | **본 프로젝트 SPDX = Apache-2.0** |
| `docs/coding-style.md` | Go / Vue / 명명 규약 |
| `docs/contributing.md` | 의존성 / 커밋 / PR / 문서 거버넌스 |
| `docs/glossary.md` | 도메인 용어 |
| `.claude/agent-roles.md` | AI별 위임 매트릭스 (dispatcher용) |
| Coroot 공식 문서 | https://docs.coroot.com (브라우저 열람만, clone 금지) |

---

## 10. 변경 규칙 (요약)

- 본 파일 변경: PR 필수 + 모듈 오너 전원 리뷰 + ADR 기록
- 상세 거버넌스: `docs/contributing.md` §4

---

**문서 버전**: v1.1 (Slim Router) · **최종 갱신**: 2026-05-18

> Quickstart로 답이 안 나오는 경우에만 사용자에게 질문.
