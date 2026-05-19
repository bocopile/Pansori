# ADR-0002: Kubernetes 기능 상태 Canonical 표

| 메타 | 값 |
|---|---|
| **상태** | Accepted |
| **결정일** | 2026-05-18 |
| **결정자** | SubOrchestrator 검증 결과 채택 |
| **관련 ADR** | ADR-0001 (K8s 버전 정책) |
| **유효 시점** | 2026-05-18 (이 표의 모든 데이터는 이 날짜 기준) |

---

## 1. 컨텍스트 (Context)

본 프로젝트의 4개 핵심 문서(`CLAUDE.md`, `coroot-mapping.md`, `0001-k8s-version-policy.md`, `k8s-integration-spec.md`)는 K8s 기능의 도입/제거/GA 시점을 빈번히 인용한다. 다중 AI(Codex/Claude/Gemini) 검증 과정에서 다음과 같은 **일관된 사실 오류**가 발견되었다.

| 항목 | 잘못된 표기 (초안) | 실제 |
|---|---|---|
| Native Sidecar GA | v1.29 | **v1.33** |
| In-place Pod Resize GA | v1.33 | **v1.35** |
| DRA GA | v1.32 | **v1.34** |
| Structured Authz stable | v1.30 | **v1.32** |
| Endpoints "제거" | 제거 | **v1.33 deprecated (제거 아님)** |
| autoscaling/v2beta1 제거 | v1.26 | **v1.25** |
| OTel Profiles signal | "v1.0 GA" | **public alpha** |

원인은 **"최초 도입"·"default-on beta"·"GA"를 구분하지 않은 채 단일 버전으로 표기**한 것이다. 본 ADR은 이를 방지하기 위해 **canonical 상태표**를 단일 출처(SSOT)로 지정한다.

---

## 2. 결정 (Decision)

### 2.1 표기 규칙 (반드시 준수)

모든 문서는 K8s 기능을 인용할 때 다음 3축을 구분해 적는다:

```
<기능명> (alpha v1.A → beta v1.B → default-on v1.C → GA v1.G)
```

- **alpha**: 최초 도입. feature gate off-by-default.
- **beta**: feature gate on-by-default 또는 명시.
- **default-on beta**: API/동작이 기본 활성화되었지만 spec 변경 가능.
- **GA (stable)**: API 안정. 하위 호환 보장.

생략 가능한 단계는 생략하되, **GA 버전은 반드시 명시**한다.

### 2.2 인용 사례

✅ Good:
> Native Sidecar Container (v1.28 alpha → v1.29 default-on beta → **v1.33 GA**)

✅ Acceptable:
> Native Sidecar Container (**v1.33 GA**, v1.29 default-on beta)

❌ Bad:
> Native Sidecar (v1.29 GA)  ← 잘못된 GA 시점

### 2.3 본 표가 단일 출처
- 4개 문서에서 K8s 기능을 인용할 때는 **본 표를 검증 기준으로** 사용한다.
- 표 갱신은 본 ADR의 PR로만 수행하며, 갱신 시 4개 문서를 동시 검토한다.
- 본 표의 "검증 상태" 컬럼이 `unverified`인 항목은 **인용을 보류**한다.

---

## 3. Canonical 기능 상태표

### 3.1 활용 (Use) — GA 또는 stable beta

| 기능 / API | alpha | beta (default) | GA (stable) | 본 프로젝트 사용 | 검증 |
|---|---|---|---|---|---|
| **Native Sidecar Containers** (initContainer + restartPolicy=Always) | v1.28 | v1.29 (default-on) | **v1.33** | T1 — 에이전트 표준 | ✅ k8s blog/docs |
| **Gateway API v1** (`gateway.networking.k8s.io/v1`) | 2022 | 2023-04 (v0.6 beta) | **2023-10-31** (v1.0) | T2 — capability-gated | ✅ k8s blog 2023-10 |
| **ValidatingAdmissionPolicy** (`admissionregistration.k8s.io/v1`) + CEL | v1.26 | v1.28 | **v1.30** | T1 — 정책 검증 | ✅ k8s docs |
| **In-place Pod Resize** (`pods/resize` subresource) | v1.27 | v1.32 → v1.33 (default-on) | **v1.35** | T1 — 동적 리소스 | ✅ k8s blog 2025-12 |
| **DRA - Dynamic Resource Allocation** (`resource.k8s.io/v1`) | v1.26 | v1.32 (구조화 파라미터) | **v1.34** | T2 — capability-gated | ✅ k8s blog 2025-09 |
| **KMS v2** | v1.25 | v1.27 | **v1.29** | T2 — etcd 암호화 검증 | ✅ k8s docs |
| **Structured Authorization Config** | v1.29 | v1.30 (default-on) | **v1.32** | T2 — 사내 IdP 외부 인가 | ✅ k8s docs v1.32 |
| **EndpointSlice** (`discovery.k8s.io/v1`) | v1.16 | v1.17 | **v1.21** | 필수 — Endpoints 대체 | ✅ k8s docs |
| **CronJob** (`batch/v1`) | v1.4 (extensions) | — | **v1.21** | 필수 — v1beta1 대체 | ✅ k8s docs |
| **HPA v2** (`autoscaling/v2`) | — | — | **v1.23** | 필수 — v2beta 대체 | ✅ k8s docs |
| **Pod Security Admission (PSA)** | v1.22 | v1.23 | **v1.25** | 필수 — PSP 대체 | ✅ k8s docs |

### 3.2 사용 금지 (Forbidden) — 제거 또는 deprecated

| API | 상태 | 시점 | 대체 |
|---|---|---|---|
| `policy/v1beta1 PodSecurityPolicy` | **제거** | **v1.25** | PSA |
| `batch/v1beta1 CronJob` | **제거** | **v1.25** | `batch/v1 CronJob` |
| `autoscaling/v2beta1` | **제거** | **v1.25** | `autoscaling/v2` |
| `autoscaling/v2beta2` | **제거** | **v1.26** | `autoscaling/v2` |
| `extensions/v1beta1 *` (전체) | **제거** | v1.16~v1.22 사이 단계적 | 해당 GA 그룹 |
| In-tree cloud-providers (AWS/Azure/GCE 등) | **제거** | **v1.31** | external CCM (On-Prem 무관) |
| `core/v1 Endpoints` | **deprecated** | **v1.33** | EndpointSlice (제거는 아님, 즉시 전환 권장) |
| `core/v1 ComponentStatus` | **long-deprecated** | (오래전) | `/healthz` 직접 폴링, kube-state-metrics |

### 3.3 옵션 (Optional, Capability-Gated)

> 본 프로젝트는 다음 기능을 **API discovery로 확인 후 활성화**. 부재 시 fallback 동작.

| 기능 | 활성화 조건 | 부재 시 Fallback |
|---|---|---|
| Gateway API v1 | `gateway.networking.k8s.io/v1` CRD 존재 | `networking.k8s.io/v1 Ingress` |
| DRA | `resource.k8s.io/v1` API 존재 | device plugin + DCGM Exporter |
| ValidatingAdmissionPolicy | `admissionregistration.k8s.io/v1` API 존재 | ValidatingWebhook 또는 비활성 |
| In-place Pod Resize | `pods/resize` 서브리소스 응답 | 재시작 기반 리사이즈 |
| Structured Authorization | kube-apiserver 플래그 (관측만) | 기존 RBAC만 |
| eBPF fentry/fexit | kernel BTF + 함수 존재 | kprobe fallback |

### 3.4 도입 보류 (Hold)

| 기능 | 현재 상태 (2026-05-18) | 도입 조건 |
|---|---|---|
| **OpenTelemetry Profiles signal (4th signal)** | **public alpha** (spec status: development) | Spec stable + Collector 안정 GA 이후 재평가 |
| **kubelet image pull credentials (Service Account 기반)** | alpha/beta (KEP-4412 등) | GA 이후 재평가, 그 전엔 사내 레지스트리 비밀 표준만 사용 |
| **WASM 기반 검사 룰 (자체)** | 자체 R&D 필요 | P5+ 별도 ADR |

---

## 4. 의존성 라이브러리 매핑 표

| Kubernetes | client-go / api / apimachinery | controller-runtime | gateway-api client |
|---|---|---|---|
| v1.36 (primary) | **v0.36.x** | **v0.24.x** | **v1.2+** |
| v1.35 | v0.35.x | v0.23.x | v1.2+ |
| v1.34 (min) | v0.34.x (또는 v0.36 ±2 호환) | v0.22.x | v1.1+ |

> 본 프로젝트는 **v0.36.x + v0.24.x** 쌍 고정 (single supported pair).
> v1.34 클러스터에서 client-go v0.36은 **공식 보장 범위 밖**(±1만 공식). 통상 동작은 하지만, 본 프로젝트는 v1.34/v1.35/v1.36 매트릭스를 **CI E2E로 직접 검증**해 동작을 보강한다.
> 출처: https://github.com/kubernetes/client-go (compatibility matrix)

### 4.1 비K8s 라이브러리

| 라이브러리 | 권장 minor pin (2026-05-18) | 비고 |
|---|---|---|
| `github.com/cilium/ebpf` | **v0.21.x** | 최신 안정. v0.16+는 너무 느슨 — exact minor pin 사용 |
| `github.com/ClickHouse/clickhouse-go/v2` | latest v2.x | v3 등장 시 별도 ADR |
| `github.com/jackc/pgx/v5` | latest v5.x | |
| `github.com/prometheus/client_golang` | latest v1.x | |

---

## 5. 갱신 절차 (Governance)

1. 본 표의 어떤 한 줄이라도 잘못된 것을 발견하면 **즉시 PR을 열어 정정**한다.
2. 정정 PR은 다음을 포함해야 한다:
   - 변경된 줄
   - 근거 URL (k8s blog, docs feature-state, KEP)
   - 본 ADR을 인용하는 4개 문서의 동기화 확인
3. K8s 릴리스(예정: v1.37 — 2026-08, v1.38 — 2026-12)마다 본 표를 일괄 재검토한다.
4. ADR-0001의 minimum 버전 상승 시 본 표의 "alpha/beta" 컬럼 중 EOL된 줄을 정리.

---

## 6. 결과 (Consequences)

### 6.1 긍정적
- 다중 AI가 작성한 문서 간 사실 일관성 확보
- 신규 작업 시 본 표만 참조하면 GA 시점 오류 방지
- 인용 형식 표준화 → 검토 비용 감소

### 6.2 부정적
- 표 유지보수 부담 (K8s 릴리스마다 검토)
- "검증되지 않은 정보 인용 보류" 규칙 위반 시 문서 작성 지연

### 6.3 완화
- CI에 `pluto` 통합해 deprecated API 자동 감지
- 분기별 K8s changelog 리뷰 의례화

---

## 7. 공식 출처 (References)

- Kubernetes Releases: https://kubernetes.io/releases/
- Version Skew Policy: https://kubernetes.io/releases/version-skew-policy/
- API Deprecation Guide: https://kubernetes.io/docs/reference/using-api/deprecation-guide/
- Sidecar Containers (stable v1.33): https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/
- In-place Pod Resize GA (v1.35): https://kubernetes.io/blog/2025/12/19/kubernetes-v1-35-in-place-pod-resize-ga/
- DRA stable (v1.34): https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/
- Endpoints deprecation (v1.33): https://kubernetes.io/blog/2025/04/24/endpoints-deprecation/
- ValidatingAdmissionPolicy: https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/
- KMS v2: https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/
- In-tree cloud-providers removal (v1.31): https://kubernetes.io/blog/2024/08/13/kubernetes-v1-31-release/
- Structured Authorization (v1.32 docs): https://v1-32.docs.kubernetes.io/docs/reference/access-authn-authz/authorization/
- Gateway API v1.0 announcement: https://kubernetes.io/blog/2023/10/31/gateway-api-ga/
- client-go versioning: https://github.com/kubernetes/client-go
- controller-runtime releases: https://github.com/kubernetes-sigs/controller-runtime/releases
- cilium/ebpf releases: https://github.com/cilium/ebpf/releases
- OpenTelemetry Profiles status: https://opentelemetry.io/status/

---

## 8. 변경 이력

| 일자 | 변경 |
|---|---|
| 2026-05-18 | 초안 채택 (Codex 웹 검색 기반 사실 검증 결과 반영) |
