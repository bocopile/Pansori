# SPEC: Kubernetes v1.34+ Integration

| 메타 | 값 |
|---|---|
| **상태** | Draft |
| **작성일** | 2026-05-18 |
| **오너** | TBD |
| **관련 ADR** | `docs/30-adr/0001-k8s-version-policy.md` |
| **영향 모듈** | `modules/agent-cluster`, `modules/agent-node`, `modules/api`, `modules/auth`, `deploy/helm`, `deploy/operator` |

---

## 1. 목적

본 프로젝트의 모든 모듈이 Kubernetes v1.34+ 환경에서 일관되게 동작하도록, **활용 API 매트릭스**, **배포 패턴**, **검증 전략**을 정의한다. 구버전 호환 코드 도입을 사전 차단하고, v1.34+ 신규 API의 가치를 실제 기능으로 연결한다.

---

## 2. 범위 (Scope)

### 2.1 포함
- Kubernetes Core API 사용 정책 (v1.34+ 기준)
- 신규 GA 기능(Native Sidecar, Gateway API, DRA, In-place Resize, ValidatingAdmissionPolicy) 활용 설계
- 배포 매니페스트/Helm/Operator 표준
- 클라이언트 라이브러리 버전 정책
- CI 검증 매트릭스

### 2.2 제외
- 비-K8s 환경(VM/베어메탈) 배포 — `docs/20-specs/vm-deployment-spec.md`
- 멀티클러스터 페더레이션 — 추후 별도 ADR/SPEC

---

## 3. 활용 API 매트릭스

### 3.1 사용 허용 (Use)

| API | 버전 | 용도 | 사용 모듈 |
|---|---|---|---|
| `core/v1 Pod, Service, ConfigMap, Secret, Node, Namespace` | stable | 기본 리소스 | agent-cluster, operator |
| `apps/v1 Deployment, DaemonSet, StatefulSet` | stable | 워크로드 관리 | operator |
| `discovery.k8s.io/v1 EndpointSlice` | stable | 서비스 엔드포인트 디스커버리 | agent-cluster |
| `networking.k8s.io/v1 NetworkPolicy` | stable | 네트워크 정책 분석 | processing/topology |
| `networking.k8s.io/v1 Ingress` | stable (legacy) | 기존 Ingress 수집만 | agent-cluster (read-only) |
| `gateway.networking.k8s.io/v1 Gateway, HTTPRoute, GRPCRoute` | v1 GA | **L7 토폴로지 1차 수집원** | agent-cluster, processing/topology |
| `batch/v1 Job, CronJob` | stable | 배치 워크로드 | agent-cluster |
| `autoscaling/v2 HPA` | stable | HPA 메트릭 분석 | processing/inspections |
| `policy/v1 PodDisruptionBudget` | stable | 가용성 분석 | processing/inspections |
| `coordination.k8s.io/v1 Lease` | stable | Leader election | operator, agent-cluster |
| `admissionregistration.k8s.io/v1 ValidatingAdmissionPolicy, ValidatingAdmissionPolicyBinding` | v1 GA | SLO/정책 검증 (옵션) | operator |
| `resource.k8s.io/v1 ResourceClaim, DeviceClass` | **v1.34 GA** (v1.32 beta) | GPU/가속기 관측 (옵션, capability-gated) | agent-cluster, processing |
| `metrics.k8s.io/v1beta1` | stable-de-facto (장기 beta) | 노드/팟 메트릭 보조 — metrics-server 설치 전제 | agent-cluster |
| `events.k8s.io/v1 Event` | stable | 이벤트 수집 | agent-cluster |
| `apiextensions.k8s.io/v1 CRD` | stable | 자체 CRD 정의 | operator |

### 3.2 사용 금지 (Forbid)

| API | 사유 |
|---|---|
| `policy/v1beta1 PodSecurityPolicy` | v1.25 제거 |
| `batch/v1beta1 CronJob` | v1.25 제거 |
| `autoscaling/v2beta1` | v1.25 제거 |
| `autoscaling/v2beta2` | v1.26 제거 |
| `core/v1 Endpoints` (직접 watch) | **v1.33 deprecated** — 제거 아님이나 EndpointSlice로 즉시 전환 |
| `core/v1 ComponentStatus` | deprecated, 곧 제거 |
| `extensions/v1beta1 *` | 모두 제거됨 |
| In-tree cloud-provider 코드 | v1.31 제거, On-Prem 무관 |

### 3.3 조건부 사용

| API | 조건 |
|---|---|
| `networking.k8s.io/v1 Ingress` | 클러스터에 Gateway API 미설치 시 fallback. 신규 기능은 Gateway API로만 |
| `node.k8s.io/v1 RuntimeClass` | 다중 런타임(예: kata, gVisor) 환경에서만 활용 |

---

## 4. 핵심 신규 기능 활용 설계

### 4.1 Native Sidecar (v1.33 GA, v1.29 default-on beta)
**사용처**: `modules/agent-node` 의 일부 컴포넌트(예: 로그 sidecar), `modules/operator` 의 자동 주입

**설계**:
```yaml
spec:
  initContainers:
    - name: coroot-telemetry-sidecar
      restartPolicy: Always           # ← Native Sidecar 표시
      image: coroot-sidecar:1.0
      lifecycle:
        preStop:
          exec: { command: ["/bin/coroot-sidecar", "drain"] }
  containers:
    - name: app
      ...
```

**금기**:
- 기존 `sidecar.istio.io/inject` 스타일 어노테이션 워크어라운드 도입 금지
- `shareProcessNamespace: true` 핵 금지

---

### 4.2 Gateway API (v1 GA)
**사용처**: `modules/processing/topology` 의 L7 토폴로지 1차 수집원

**책임**:
- `HTTPRoute` / `GRPCRoute` 로부터 외부 진입 트래픽 흐름 매핑
- `Gateway` 리스너 기준 SLO 측정 단위 정의
- 기존 Ingress가 있으면 보조 수집

**설계 포인트**:
- `gateway.networking.k8s.io/v1` 만 watch (실험적 v1alpha2/v1beta1 watch 금지)
- HTTPRoute의 `backendRefs[*].name`을 Service Map의 엣지로 매핑
- `ParentRef`가 없는 고아 라우트 감지 → inspections 신호

---

### 4.3 ValidatingAdmissionPolicy + CEL (v1.30 GA)
**사용처**: `modules/operator` 의 자체 정책 + 사용자 정의 SLO 정책

**적용 패턴**:
- 사내 SLO 룰을 CEL 표현식으로 정의 → 위반 워크로드의 admission 거부 또는 경고
- 예: "모든 user-facing Pod은 LivenessProbe + ReadinessProbe 필수"

**금기**:
- ValidatingWebhookConfiguration 신규 도입 금지 (CEL로 대체 가능한 경우)
- 자체 webhook 서버는 CEL로 표현 불가한 케이스(외부 데이터 조회 등)에만

---

### 4.4 In-place Pod Resize (v1.35 GA, v1.33 default-on beta)
**사용처**: `modules/agent-node` (DaemonSet), `modules/operator` 의 자동 튜닝

**설계**:
- 에이전트는 자체 리소스 사용량을 추적해 `resize` 서브리소스로 동적 조정
- `kubectl patch pod ... --subresource resize ...`
- 재시작 없는 메모리 증설 → DaemonSet 가용성 유지

---

### 4.5 DRA (Dynamic Resource Allocation, v1.34 GA, v1.32 beta)
**사용처**: `modules/agent-cluster` + `modules/processing/topology`

**책임**:
- `ResourceClaim` / `DeviceClass` 를 watch하여 GPU/가속기 사용 워크로드 식별
- 사내 LLM 추론 서비스의 GPU 활용도/메모리 메트릭 1차 시민으로 노출
- 노드 토폴로지 그래프에 가속기 노드 별도 표시

---

### 4.6 KMS v2 (v1.29 GA)
**사용처**: `modules/storage/meta` 의 시크릿 암호화 정책 검증 (operator가 권고)

**책임**:
- etcd 암호화 설정이 KMS v2 기반인지 확인 (사내 KMS와 통합 권장)
- KMS v1 사용 시 inspection 경고

---

## 5. 클라이언트 라이브러리 정책

| 라이브러리 | 버전 | 비고 |
|---|---|---|
| `k8s.io/client-go` | `v0.36.x` | K8s v1.36 매칭 |
| `k8s.io/apimachinery` | `v0.36.x` | |
| `k8s.io/api` | `v0.36.x` | |
| `sigs.k8s.io/controller-runtime` | `v0.24.x` | K8s v1.36 매칭 (v0.21=v1.33) |
| `sigs.k8s.io/gateway-api` | `v1.2+` | Gateway API CRD 클라이언트 |
| `github.com/cilium/ebpf` | `v0.21.x` | eBPF (CO-RE) — exact minor pin. 상세는 ADR-0002 §4.1 |

**호환성 원칙** (client-go 공식 정책 기준):
- **공식 보장**: 클라이언트 버전과 API 서버 버전 **차이는 ±1 마이너** 까지
- **통상 동작 (보장 외)**: ±2 마이너까지 *대부분 동작하지만 공식 보장은 아님*. v1.34 클러스터에서 `client-go v0.36`은 이 범위에 해당
- 본 프로젝트는 **v0.36 + K8s v1.34~v1.36** 매트릭스를 E2E로 직접 검증 (CI에서 ±2 동작을 보강 보장)
- v1.37 출시 시 `client-go v0.37`로 갱신, 최소 지원은 자동으로 v1.35로 상승
- 출처: https://github.com/kubernetes/client-go

---

## 6. RBAC 최소 권한 원칙

### 6.1 agent-node (DaemonSet)
```yaml
rules:
  - apiGroups: [""]
    resources: ["nodes", "pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["get", "list", "watch"]
```

### 6.2 agent-cluster (Deployment, 1개)
- 위 + `apps/*` (read), `batch/*` (read), `networking.k8s.io/*` (read), `gateway.networking.k8s.io/*` (read), `resource.k8s.io/*` (read), `events.k8s.io/*` (read)
- write 권한 절대 부여 금지

### 6.3 operator
- 자체 CRD에 대해서만 RW
- `admissionregistration.k8s.io/*` (write — 정책 생성 시)
- Pod resize 서브리소스 `pods/resize` (write — 자동 튜닝 시)

**금기**:
- ❌ `cluster-admin` 부여
- ❌ wildcard `*` resource
- ❌ secret 전체 watch (지정 namespace의 운영용 secret만)

---

## 7. 배포 표준

### 7.1 Helm Chart (`deploy/helm/`)
- Helm v3.16+ 만 지원
- `Chart.yaml` `kubeVersion: ">=1.34.0-0"` 명시
- values 스키마: JSON Schema(`values.schema.json`)로 검증
- subchart 의존 최소화 — Prometheus, ClickHouse는 외부 차트 사용 시 dependency로만

### 7.2 Operator (`deploy/operator/`)
- `controller-runtime v0.24.x` 기반
- CRD 그룹: `coroot.local/v1alpha1` (네임스페이스: 사내 도메인)
- 단일 CRD `Coroot` (전체 컴포넌트 배포 단위)
- Reconcile은 idempotent, 외부 시스템 변경 전 dry-run 후 적용

### 7.3 멀티 배포 모드
| 모드 | 매니페스트 |
|---|---|
| K8s + Operator | `deploy/operator/` |
| K8s + Helm only | `deploy/helm/` |
| VM/베어메탈 | `deploy/systemd/` (별도 SPEC) |
| Air-gapped K8s | Helm + 사내 레지스트리 이미지 미러 |

---

## 8. 검증 / CI

### 8.1 단위/통합 테스트
- `envtest` (controller-runtime) 로 API 시뮬레이션
- `fake.NewSimpleClientset` 사용 시 객체 라이프사이클 명시

### 8.2 E2E 매트릭스
| 항목 | 값 |
|---|---|
| K8s 버전 | v1.34, v1.35, **v1.36 (primary)** |
| 배포 도구 | kind, k3s (RKE2 옵션) |
| CNI | Cilium (1차), Calico (보조 검증) |
| Container Runtime | containerd 1.7+ (Docker 미지원) |

### 8.3 Conformance
- `kubectl-validate`로 매니페스트 검증
- Gateway API conformance 테스트 (해당 부분만)
- `pluto`로 deprecated API 사용 차단 (CI 게이트)

---

## 9. 마이그레이션 / 호환

### 9.1 신규 작성 시 회피해야 할 안티패턴 (Anti-patterns)

> 본 절은 **포팅 가이드가 아니다**. ADR-0003에 따라 Coroot 코드를 본 저장소에 들이지 않는다. 본 표는 신규 작성 시 흔히 빠지는 함정을 사전 차단하기 위한 안티패턴 목록이다.

| 잘못된 패턴 (사용 금지) | 올바른 패턴 |
|---|---|
| `corev1.Endpoints` watch | `discoveryv1.EndpointSlice` watch (v1.33 deprecated) |
| `policyv1beta1.PodSecurityPolicy` import | PSA (Pod Security Admission)만 사용 |
| `batchv1beta1.CronJob` import | `batchv1.CronJob` (v1.25 제거) |
| `cloudprovider.Interface` import | 사용 금지 — On-Prem 무관 |
| 수동 sidecar 어노테이션 / `shareProcessNamespace` 핵 | Native Sidecar (v1.33 GA, `initContainers[].restartPolicy: Always`) |
| `bcc.BPF(...)` / in-pod 컴파일 | `cilium/ebpf` CO-RE v0.21.x |

자세한 안티패턴은 `CLAUDE.md` §4 참조.

### 9.2 Graceful Degradation
다음 기능은 클러스터 capability 확인 후 활성화:
- Gateway API → CRD 존재 확인 (`gateway.networking.k8s.io`)
- DRA → `ResourceClaim` API 존재 확인
- ValidatingAdmissionPolicy → API 존재 확인
- In-place Resize → feature gate 확인 (`InPlacePodVerticalScaling`)

미지원 시: 해당 기능 비활성화, UI에 "클러스터 capability 부족" 표시.

---

## 10. 미해결 사항 (Open Questions)

- [ ] OpenShift 4.18+ 호환 검증을 v1로 포함할지 (별도 ADR로 분리 권장)
- [ ] Sidecar 자동 주입 시 namespace 라벨 vs 워크로드 어노테이션 — 어느 게 디폴트?
- [ ] DRA의 `DeviceClass` 라벨 컨벤션 사내 표준 필요
- [ ] CEL 정책 라이브러리를 자체 패키지로 제공할지 (`pkg/cel/`?)
- [ ] kubelet 인증을 어떻게 표준화할지 (CSR 자동승인 정책)

---

## 11. 후속 산출물

- 본 SPEC 채택 후:
  1. `modules/operator/api/v1alpha1/` — **kubebuilder Go struct로 직접 CRD 정의** (contracts-spec §2 결정에 따라 proto 미사용)
  2. `modules/agent-cluster/internal/discovery/` — EndpointSlice + Gateway API 워치 구현
  3. `modules/operator/internal/sidecar/` — Native Sidecar 자동 주입 (옵션)
  4. `deploy/helm/values.schema.json` — 스키마 정의
  5. `.github/workflows/k8s-matrix.yml` (또는 사내 CI 등가물) — v1.34/35/36 매트릭스
