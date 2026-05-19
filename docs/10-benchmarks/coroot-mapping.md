# Coroot ↔ 우리 프로젝트 매핑 (Benchmark Mapping)

> **목적**: Coroot 오픈소스의 컴포넌트를 우리 On-Premise 프로젝트의 모듈로 1:1 대응시켜,
> AI에게 코드 생성을 요청할 때 "Coroot의 X를 참고해 우리의 Y를 만들어줘"가 즉시 통하도록 만든다.
>
> **작성일**: 2026-05-18
> **베이스 스택**: Go 1.26 + Vue 3 (Coroot 동일)
> **타겟 환경**: On-Premise (사내망 + 제한적 외부 게이트웨이)
> **Kubernetes 정책**: **v1.34 minimum / v1.36 primary** (ADR-0001 참조)

---

## 0. 표기 규약

| 표기 | 의미 |
|---|---|
| 🟢 **Copy** | ❌ **사용 금지** (ADR-0003) — Coroot 코드 카피·포팅 불가 |
| 🟡 **Adapt** | ❌ **사용 금지** (ADR-0003) — 코드 차용 불가 |
| 🔵 **Inspire** | 패턴만 참고, 코드는 처음부터 |
| 🔴 **Replace** | On-Prem 부적합 → 대체 구현 |
| ⚫ **Drop** | 사용 안 함 |

---

## 1. 아키텍처 레이어별 매핑

### 1.1 Collection Layer (데이터 수집)

| Coroot 원본 | 우리 모듈 (가칭) | 채택 깊이 | On-Prem 변경 사항 |
|---|---|---|---|
| `coroot/coroot-node-agent` (eBPF 노드 에이전트) | `modules/agent-node/` | 🔵 **Inspire** | - 외부 텔레메트리 export 제거<br>- 인증서 사내 PKI 연동<br>- proxy 설정 추가<br>- **bcc 미사용 → cilium/ebpf CO-RE 채택**<br>- **커널 5.10+ 가정 (fentry/fexit)** |
| `coroot/coroot-cluster-agent` (K8s/DB 수집) | `modules/agent-cluster/` | 🔵 **Inspire** | - K8s 전용 → VM/베어메탈 동시 지원<br>- AWS SDK 미사용<br>- **EndpointSlice 채택**<br>- **Gateway API watch 추가**<br>- **DRA ResourceClaim watch 추가** |
| `coroot/coroot-pg-agent` | `modules/agent-db-postgres/` | 🔵 **Inspire** | 사내 PG 인증 (Kerberos/LDAP) 옵션 추가. `pg_stat_statements` 수집은 공식 PostgreSQL docs 기반 |
| `coroot/coroot-mysql-agent` | `modules/agent-db-mysql/` | 🔵 **Inspire** | 동일 — MySQL `performance_schema` 공식 docs 기반 |
| `coroot/coroot-aws-agent` | — | ⚫ **Drop** | On-Prem이므로 불필요 |
| (없음) | `modules/agent-onprem-vm/` | 🔵 **Inspire** | systemd 서비스 수집기 (Coroot에는 없음) |

**핵심 라이브러리 후보 (Go)**:
- eBPF: `github.com/cilium/ebpf` 또는 `github.com/aquasecurity/libbpfgo`
- OTLP export: `go.opentelemetry.io/otel/exporters/otlp/*`
- Prom export: `github.com/prometheus/client_golang`

---

### 1.2 Transport Layer (전송)

| Coroot 원본 | 우리 모듈 | 채택 | 변경 사항 |
|---|---|---|---|
| OTLP gRPC/HTTP | `modules/transport/otlp/` | 🔵 **Inspire** | OpenTelemetry 공식 spec(Apache-2.0)에서 직접 구현. TLS 사내 CA 연결 |
| Prometheus Remote Write | `modules/transport/promrw/` | 🔵 **Inspire** | Prometheus 공식 spec에서 직접 구현. 동일 |
| (자체 프로토콜 없음) | `modules/transport/internal/` | 🔵 **Inspire** | 에이전트↔서버 컨트롤 채널 (필요시) |

---

### 1.3 Storage Layer (저장)

| Coroot 원본 | 우리 모듈 | 채택 | 변경 사항 |
|---|---|---|---|
| Prometheus (TSDB) | `modules/storage/metrics/` (래퍼) | 🔵 **Inspire** | 공식 client 라이브러리(Apache-2.0) 사용. 원격 Prom 클러스터 연결 추상화 |
| ClickHouse (로그/트레이스/프로파일) | `modules/storage/telemetry/` (래퍼) | 🔵 **Inspire** | clickhouse-go(Apache-2.0) 사용. 다중 샤드/리플리카 사내 지원 |
| PostgreSQL/SQLite (메타데이터) | `modules/storage/meta/` | 🔵 **Inspire** | pgx(MIT) 사용. 운영DB는 PG 강제 (SQLite는 dev only) |

**On-Prem 핵심 결정**:
- ClickHouse는 사내 배포 가능 → 그대로 활용
- 외부 SaaS(Grafana Cloud 등) 의존 없음 확인

---

### 1.3.5 Kubernetes Integration Layer (★ v1.34+ 정책)

본 프로젝트의 모든 K8s 통합 코드는 **v1.34 minimum / v1.36 primary** 정책을 따른다.
(상세: ADR-0001, `k8s-integration-spec.md`)

#### A. 제거할 항목 (Coroot 원본에서 빼야 할 것)

| Coroot 원본 패턴 | 제거 사유 | 대체 |
|---|---|---|
| `policyv1beta1.PodSecurityPolicy` 처리 | K8s v1.25 제거됨 | PSA (Pod Security Admission) |
| `batchv1beta1.CronJob` | v1.25 제거 | `batchv1.CronJob` |
| `autoscaling/v2beta1` | v1.25 제거 | `autoscaling/v2` |
| `autoscaling/v2beta2` | v1.26 제거 | `autoscaling/v2` |
| `corev1.Endpoints` 직접 watch | EndpointSlice가 표준 | `discoveryv1.EndpointSlice` |
| In-tree cloud-provider 코드 | v1.31 제거 + On-Prem | 삭제 |
| `corev1.ComponentStatus` | deprecated | `/healthz` 직접 폴링 |
| 수동 sidecar(initContainer+sleep) | 안티 패턴 | Native Sidecar (**v1.33 GA**) |
| `bcc` 기반 eBPF in-pod 컴파일 | CO-RE 불호환 | `cilium/ebpf` (CO-RE) |
| 커널 4.x 호환 워크어라운드 | 5.10+ 보장 | fentry/fexit 우선, kprobe fallback |
| OTel SDK v0.x 호환 | v1.0 GA (로그/메트릭) | OTel v1.x 만 사용 |

#### B. 새로 추가할 v1.34+ 기능 (Coroot에 없거나 약한 부분)

> **Tier 정의 (재정의)**:
> - **T1 (필수, hard requirement)** = 모든 운영 클러스터에서 활성화. 클러스터가 미지원이면 도입 불가. P1~P3 내 구현.
> - **T2 (옵션, capability-gated)** = API discovery로 capability 확인 후 활성화. 부재 시 명시된 fallback 동작. P3~P5 내 구현.
> - **T3 (보류 / R&D)** = 표준 안정 GA 또는 사내 우선순위 확정 후 별도 ADR. P6+.

##### T1 — 필수 (capability gating 불필요)

| 신규 요소 | 모듈 매핑 | 핵심 가치 |
|---|---|---|
| **Native Sidecar (v1.33 GA, v1.29 default-on beta)** | `modules/operator/internal/sidecar/` | 에이전트 배포 표준화 — v1.34 minimum이라 GA 보장 |
| **ValidatingAdmissionPolicy + CEL (v1.30 GA)** | `modules/operator/internal/policy/` | SLO 정책을 CEL로 |
| **EndpointSlice 사용** | `modules/agent-cluster` | Endpoints는 v1.33 deprecated |

##### T2 — 옵션 (capability 확인 후 활성화, fallback 동봉)

| 신규 요소 | 모듈 매핑 | Capability 체크 | **Fallback** |
|---|---|---|---|
| **In-place Pod Resize (v1.35 GA, v1.33 default-on beta)** | `modules/agent-node` + `operator` | `pods/resize` 서브리소스 응답 (v1.34 클러스터는 beta default-on이지만 GA semantic 보장 안 됨) | **재시작 기반 리사이즈** |
| **Gateway API v1 (2023-10 GA)** | `agent-cluster` + `processing/topology` | `gateway.networking.k8s.io/v1` CRD 존재 | `networking.k8s.io/v1 Ingress` 수집 + L7 메타 약화 |
| **DRA (v1.34 GA)** | `agent-cluster` + `processing` | `resource.k8s.io/v1` API 존재 | **device plugin + DCGM/HabanaLabs Exporter** (PromRW 수집) |
| **KMS v2 (v1.29 GA) 정책 검증** | `processing/inspections` | etcd encryption config 조회 권한 | 비활성 (검사 항목 누락 표시) |
| **Structured Authorization (v1.32 stable)** | `processing/inspections` | kube-apiserver flag 관측 | 기존 RBAC 정책만 검사 |
| **Hubble/Cilium L4-L7 통합** | `agent-cluster` | Cilium 설치 확인 (CRD) | eBPF 기반 자체 수집만 (덜 풍부) |
| **Service Mesh 메타 (Istio Ambient/Linkerd)** | `agent-cluster` | 메시 CRD 존재 | 메시 메타 없음, 일반 L7만 |
| **Prometheus 3.x (UTF-8 라벨, Native Hist)** | `storage/metrics` | Prom 서버 버전 ≥ 3.0 | Prom 2.x 호환 모드 (UTF-8 라벨 escape) |
| **OpenCost 통합** | `processing/cost/` | OpenCost 설치 확인 | 비용 위젯 숨김 |

##### T3 — 보류 / R&D (별도 ADR)

| 신규 요소 | 모듈 매핑 | 보류 사유 | **Fallback / 임시 대체** |
|---|---|---|---|
| **OTel Profiles signal** | `transport/otlp` + `processing/profile` | 2026-05 기준 **public alpha** (spec status: development) | Pyroscope 호환 포맷 또는 pprof 자체 포맷 |
| **WASM 기반 검사 룰** | `processing/inspections/wasm/` | 사용자 정의 룰 모델 미정 | CEL 기반 룰 + YAML DSL |
| **AI/ML 워크로드 관측 전용 뷰** | `processing/ml/` | 도메인 요구 확정 전 | DRA fallback 데이터로 일반 워크로드 뷰 사용 |
| **Air-gapped LLM RCA (사내 vLLM/Ollama)** | `processing/rca/local/` | 사내 LLM GW 정책 확정 전 | 규칙 기반 RCA만 |
| **Cost-aware sampling** | `transport/sampling/` | 알고리즘/예산 모델 R&D 필요 | 정적 head sampling + 꼬리 샘플링 |

#### C. 클라이언트 라이브러리 (확정)

| 라이브러리 | 버전 |
|---|---|
| `k8s.io/client-go` | `v0.36.x` |
| `k8s.io/apimachinery` | `v0.36.x` |
| `sigs.k8s.io/controller-runtime` | `v0.24.x` |
| `sigs.k8s.io/gateway-api` | `v1.2.x` |
| `github.com/cilium/ebpf` | `v0.21.x` (exact minor pin — v0.16+는 너무 느슨) |

---

### 1.4 Processing Layer (분석/상관)

| Coroot 원본 | 우리 모듈 | 채택 | 변경 사항 |
|---|---|---|---|
| Service Map 구축 | `modules/processing/topology/` | 🔵 **Inspire** | Coroot 공식 docs의 토폴로지 컨셉 학습 후 자체 구현. 도메인 라벨링 룰 사내 컨벤션 |
| Inspections 엔진 (anomaly/SLO) | `modules/processing/inspections/` | 🔵 **Inspire** | Coroot docs의 inspection 카테고리 학습 후 자체 구현. 사내 SLO 정책 룰셋 |
| Deployment Diff | `modules/processing/deployments/` | 🔵 **Inspire** | 사내 배포 시스템(예: ArgoCD/Spinnaker) 연동 |
| **AI RCA (LLM 기반)** | `modules/processing/rca/` | 🔴 **Replace** | **사내 LLM 게이트웨이 또는 규칙기반 RCA로 대체** |
| `logparser` (Drain 알고리즘) | `modules/processing/logparser/` | 🔵 **Inspire** | **Drain 알고리즘은 IEEE ICWS 2017 논문(He et al.)에서 직접 구현**. 한국어 로그 패턴 보강 |

**🔴 RCA 대체 전략 (중요)**:
- 1차: Coroot의 inspections 결과를 규칙 기반으로 우선순위화
- 2차: 사내 LLM 게이트웨이(있다면) 호출 — 프롬프트 템플릿만 외부화
- 절대 외부 OpenAI/Anthropic 직접 호출 ❌

---

### 1.5 Presentation Layer (UI/API)

| Coroot 원본 | 우리 모듈 | 채택 | 변경 사항 |
|---|---|---|---|
| Vue 3 SPA | `modules/ui/` | 🔵 **Inspire** | Vue 3 공식 docs로 자체 구현. 사내 디자인 시스템 적용, i18n(ko/en) |
| REST/JSON API (Go) | `modules/api/` | 🔵 **Inspire** | OpenAPI 3.1로 자체 정의. 사내 SSO(SAML/OIDC) 연동 |
| 인증/RBAC | `modules/auth/` | 🔵 **Inspire** | OIDC/SAML 표준 RFC 직접 구현. 사내 IdP 연동, 로컬 계정 제거 |

---

## 2. 의존성 매트릭스

```
┌─────────────────────────────────────────────────────┐
│  ui/  ──────────────────► api/                       │
│                            │                          │
│                            ▼                          │
│                     processing/                       │
│                      ├─ topology/                     │
│                      ├─ inspections/                  │
│                      ├─ rca/   ◄── (사내 LLM GW)     │
│                      └─ logparser/                    │
│                            │                          │
│                            ▼                          │
│                     storage/                          │
│                      ├─ metrics/  (Prometheus)        │
│                      ├─ telemetry/(ClickHouse)        │
│                      └─ meta/     (PostgreSQL)        │
│                            ▲                          │
│                            │                          │
│                     transport/  (OTLP, PromRW)        │
│                            ▲                          │
│           ┌────────────────┼────────────────┐         │
│           │                │                │         │
│      agent-node/    agent-cluster/    agent-db-*/     │
│      (eBPF)         (K8s/VM)          (PG/MySQL)      │
└─────────────────────────────────────────────────────┘
```

---

## 3. 우선순위 (Phase 계획)

| Phase | 모듈 | 산출물 | 예상 기간 |
|---|---|---|---|
| **P0** | `docs/`, `modules/contracts/` | 스펙·인터페이스 정의 (proto/openapi), **ADR-0001 채택** | 1주 |
| **P1** | `storage/`, `transport/` | 저장·전송 래퍼 자체 구현 (공식 OTel/Prom/CH 라이브러리 사용 — ADR-0003), **OTel v1.0 메트릭/로그/트레이스** (Profile signal은 T3) | 2주 |
| **P2** | `agent-node/` | eBPF 수집기 MVP, **CO-RE only, Native Sidecar 배포** | 3주 |
| **P3** | `processing/topology` + `processing/inspections` | 서비스 맵 + 기본 알람. **Gateway API/DRA는 capability-gated (T2)**, fallback(Ingress/device-plugin)도 동시 구현 | 3주 |
| **P4** | `ui/`, `api/`, `auth/` | UI 통합 + SSO, **ValidatingAdmissionPolicy 정책 (T1)**, **OpenCost 위젯 (T2)** | 3주 |
| **P5** | `processing/rca` | 사내 LLM 게이트웨이 연동 또는 규칙기반 RCA, **WASM 검사 룰** | 2주 |
| **P6** | `agent-cluster/`, `agent-db-*` | 확장 수집기, **Hubble/Service Mesh 메타 통합** | 2주 |
| **P7+** | (선택) | **AI/ML 워크로드 관측**, **Cost-aware sampling**, Air-gapped LLM RCA | 별도 |

---

## 4. 라이선스 정책 (✅ 해소 — ADR-0003 순수 영감)

> **결정 (재결정)**: 2026-05-18, **경로 E (Pure Inspiration)** 채택. 상세: `docs/30-adr/0003-license-policy.md`.

### 4.1 결정 요약
- **Coroot 코드를 본 저장소에 두지 않는다** (clone/copy/번역/refactor 모두 금지)
- **공식 문서·논문·표준에서만** 학습 (whitelist: ADR-0003 §2.2)
- **법무 검토 불필요** — AGPL 트리거 조건 구조적으로 회피
- **본 프로젝트 SPDX**: ADR-0003-1에서 결정 (AGPL 아닌 것 중에서)

### 4.2 본 매핑표 Tier 강제 변환
ADR-0003 §2.8에 따라 모든 항목의 채택 깊이는 다음과 같이 일괄 변환된다 (별도 PR에서 §1.1~§1.5 표의 "채택 깊이" 컬럼 갱신):

| 기존 표기 | 강제 변환 결과 | 의미 |
|---|---|---|
| 🟢 **Copy** | **🔵 Inspire** | 코드 카피 불가 — 공식 문서로 컨셉 학습 후 자체 구현 |
| 🟡 **Adapt** | **🔵 Inspire** | 코드 차용 불가 — 동일 |
| 🔵 **Inspire** | 유지 | 변화 없음 |
| 🔴 **Replace** | 유지 | 변화 없음 |
| ⚫ **Drop** | 유지 | 변화 없음 |

"변경 사항" 컬럼은 이제 **공식 문서/논문에서 학습한 컨셉을 어떻게 우리 도메인에 맞게 적용할 것인가**로 의미가 재정의된다.

### 4.3 가드레일 (단순화)
- ❌ **클린룸 절차 불필요** (애초에 AGPL 코드를 안 봄)
- ❌ **experiments/ build tag 격리 불필요** (experiments/에도 AGPL 금지)
- ❌ **`.dockerignore` AGPL 제외 룰 불필요**
- ✅ **PR template에 1-line 체크**: "본 PR은 AGPL 라이선스 코드를 본 저장소에 도입하지 않았다"
- ✅ **간이 의존성 license 검사** (`go-licenses` 등): 우연한 AGPL 의존성 도입 차단 — 일반 OSS 컴플라이언스 관점

### 4.4 잔여 후속 액션 (P0 종료 전)
- [x] ADR-0003-1 작성 (배포 SPDX 결정) — 완료 (Apache-2.0 채택)
- [x] 본 §1.1~§1.5 표의 "채택 깊이" 일괄 강등 — 완료 (모두 🔵 Inspire)
- [x] 매핑표 §6 "다음 액션"의 reference/ 클론 항목 제거 — 완료
- [ ] (선택) 법무에 본 ADR을 통보용으로 공유 (검토 요청 아님, 데드라인 부담 0)

---

## 5. AI 코드 생성용 컨텍스트 카드 (예시)

각 모듈 폴더 진입 시 AI가 참고할 1-pager. 모듈마다 작성.

```markdown
# modules/agent-node/CLAUDE.md (예시)

## 이 모듈은 무엇인가
Coroot의 `coroot-node-agent`를 벤치마크한 eBPF 기반 노드 텔레메트리 수집기.

## 채택 깊이
🔵 Inspire — ADR-0003에 따라 Coroot 코드는 보지 않는다. 컨셉만 채택.

## 학습 원천 (Whitelist — ADR-0003 §2.2)
- Coroot 공식 문서: https://docs.coroot.com (Architecture · Configuration)
- libbpf / cilium/ebpf 공식 예제
- Linux kernel tracing 문서 (kprobes, fentry/fexit)
- Drain 알고리즘 논문 (IEEE ICWS 2017 — He et al.)
- OpenTelemetry spec (OTLP)

## On-Prem 제약
- 외부 SaaS 호출 금지 — 모든 export는 `transport/` 모듈 경유
- 사내 PKI 인증서 사용 — `pkg/tls/internal_ca.go` 호출
- proxy 환경변수(HTTP_PROXY/NO_PROXY) 반드시 존중

## 하지 말 것
- ❌ AWS SDK import
- ❌ 원본의 `cloud/` 패키지 포팅
- ❌ 임의의 외부 도메인 직접 호출

## 의존 모듈
- `modules/transport/otlp`
- `modules/transport/promrw`
- `modules/storage/meta` (에이전트 등록/하트비트)
```

---

## 6. 다음 액션

1. ~~**`reference/coroot-upstream/`에 Coroot 본체 fork 클론**~~ — ❌ ADR-0003에 의해 폐기 (Coroot 코드 clone 금지)
2. **`docs/20-specs/`에 모듈별 PRD 작성** — 본 매핑표 기반, 공식 문서/논문에서 학습
3. **`modules/contracts/`에 proto/openapi 정의** — SubOrchestrator가 P1에서 골격 생성
4. **루트 `CLAUDE.md`는 작성 완료** (2026-05-18, v1.1)

> 본 문서는 살아있는 문서. 모듈 작업 시작 전 채택 깊이·변경 사항이 바뀌면 즉시 갱신.
