# ADR-0001: Kubernetes 최소 지원 버전 정책

| 메타 | 값 |
|---|---|
| **상태** | Proposed (Draft) |
| **결정일** | 2026-05-18 |
| **결정자** | TBD (모듈 오너 회의 후 확정) |
| **관련 ADR** | — |
| **관련 문서** | `CLAUDE.md` §2, `docs/10-benchmarks/coroot-mapping.md`, `docs/20-specs/k8s-integration-spec.md` |

---

## 1. 컨텍스트 (Context)

본 프로젝트는 Coroot을 벤치마크하는 On-Premise 옵저버빌리티 플랫폼이다. 사내망 + 제한적 외부만 허용되는 환경에서 운영되며, 주요 배포 대상은 **Kubernetes 클러스터(kubeadm/Kubespray/RKE2/k3s)** 와 일부 VM/베어메탈이다.

2026-05-18 현재 Kubernetes 출시 상태:
- v1.36: 2026-04 (최신, current)
- v1.35: 2025-12
- v1.34: 2025-08
- v1.33: 2025-04 (지원 종료 임박)
- v1.32 이하: 커뮤니티 지원 종료

Kubernetes는 마이너 버전당 약 4개월, 총 14개월의 패치 지원(N, N-1, N-2)을 제공한다.

본 프로젝트는 신규 코드베이스이므로 **구버전 호환 부채를 가지지 않는** 출발선에서 시작할 기회가 있다. 동시에 사내 클러스터의 업그레이드 사이클(통상 6~12개월 지연)을 감안해야 한다.

---

## 2. 결정 (Decision)

### 2.1 지원 버전
- **최소 지원 버전 (MinSupported)**: Kubernetes **v1.34**
- **타깃 검증 버전 (Tested)**: v1.34, v1.35, **v1.36 (primary)**
- **권장 버전**: v1.36+
- **EOL 정책**: 새 K8s 마이너 버전 출시 후 **2개 마이너 버전** 지났을 때 해당 버전 지원 종료 (예: v1.38 출시 시 v1.35 지원 종료)

### 2.2 즉시 제거 (구버전 API/패턴)
이 ADR 채택 시점부터 코드베이스에 다음을 도입/포팅하지 않는다:

1. **PodSecurityPolicy** 처리 코드 (v1.25 제거 완료)
2. **batch/v1beta1 CronJob** (v1.25 제거)
3. **autoscaling/v2beta1** (v1.25 제거), **autoscaling/v2beta2** (v1.26 제거)
4. **In-tree cloud-provider** 의존 (v1.31 제거)
5. **ComponentStatus** API (deprecated)
6. **Endpoints** API 직접 사용 — `discovery.k8s.io/v1 EndpointSlice`만 사용
7. **수동 Sidecar 주입(initContainer + shareProcessNamespace 핵)** — Native Sidecar(**v1.33 GA**, v1.29 default-on beta) 사용
8. **PSP 마이그레이션 헬퍼** — PSA(Pod Security Admission)만 지원
9. **kubelet 4.x 호환 path** — kubelet/kubelet-config v1.34+ 가정
10. **bcc 기반 in-pod eBPF 컴파일** — CO-RE(`cilium/ebpf`)만 사용 (커널 5.10+ 가정)

### 2.3 적극 활용 (v1.34+ 표준)

> ⚠️ 모든 GA 시점은 공식 출처(`kubernetes.io/blog`, `feature state` 문서) 기준. T1 필수 vs T2/T3 옵션 분류는 `docs/10-benchmarks/coroot-mapping.md` 참조.

- **Native Sidecar Container** (**v1.33 GA**, v1.29 default-on beta): 에이전트 배포 표준
- **Gateway API v1.0** (2023-10 GA): L7 토폴로지 옵션 수집원 (Ingress fallback)
- **ValidatingAdmissionPolicy / CEL** (v1.30 GA): 정책 검증
- **In-place Pod Resize** (**v1.35 GA**, v1.33 default-on beta): 에이전트 동적 리소스 조정
- **DRA - Dynamic Resource Allocation** (**v1.34 GA**, v1.32 beta): GPU/가속기 옵저버빌리티 (옵션)
- **Structured Authorization Config** (**v1.32 stable**, v1.30 beta): 외부 인가 정책 (사내 IdP 연동)
- **KMS v2** (v1.29 GA): 시크릿 암호화 통합
- **kubelet image pull credentials**: 사내 레지스트리 인증 표준화 (구체 KEP/버전은 SPEC에서 별도 명시 — 현재 일부 alpha/beta 단계)

### 2.4 클라이언트 라이브러리
- `k8s.io/client-go` 버전: K8s **v1.36 매칭** (`v0.36.x`)
- `controller-runtime`: **v0.24.x** (K8s v1.36 호환 — v0.21은 v1.33 계열)
- `apimachinery` 호환성: v1.34~v1.36 동시 동작 시험 (e2e 테스트로 검증)

### 2.5 배포 도구
- **Helm**: v3.16+ (Helm v2 미지원)
- **Operator SDK**: `operator-sdk` v1.40+ 또는 controller-runtime 직접
- **k3s/RKE2**: v1.34 기반 이상만 지원
- **OpenShift**: v4.18+ (K8s v1.34 매칭) 검토 시 필요

---

## 3. 결과 (Consequences)

### 3.1 긍정적
- 구버전 호환 부채 0에서 출발 → 코드 단순성
- v1.34+ 신규 API(Native Sidecar, Gateway API, DRA) 활용 → 차별화
- 의존성 라이브러리 최신 버전만 — 보안/성능 이점
- 테스트 매트릭스 작음 (3개 버전만 검증)

### 3.2 부정적 / 위험
- **사내 클러스터 업그레이드 강제**: v1.33 이하 운영 중인 클러스터는 도입 불가
- 일부 보수적 운영팀의 v1.34 도입 지연 시 마찰
- DRA(v1.34 GA) 등 신규 GA 기능은 노드 OS/커널 요건 동반 (containerd 1.7+, 커널 5.10+)
- 커널 5.10+ 가정으로 일부 RHEL 8(커널 4.18) 노드 배제

### 3.3 완화책 (Mitigation)
- **Compat 매트릭스 문서화**: `docs/40-runbooks/k8s-compat-matrix.md` 유지
- **사내 클러스터 업그레이드 가이드 제공**: `docs/40-runbooks/upgrade-guide.md`
- **에이전트 OS 요건 명시**: 커널 5.10+, glibc 2.28+ (RHEL 9 / Ubuntu 22.04 / Rocky 9)
- **graceful degradation**: DRA 등 신규 기능은 클러스터에서 지원 시에만 활성화, 없을 시 비활성화

---

## 4. 검토된 대안 (Alternatives Considered)

### 대안 A: v1.30을 최소로 설정
- 장점: 더 많은 사내 클러스터 커버
- 단점: Native Sidecar(v1.33 GA), DRA(v1.34 GA), In-place Resize(v1.35 GA) 모두 미지원 → 핵심 차별화 불가
- **기각 사유**: 1년 전 버전을 minimum으로 두는 것은 신규 프로젝트의 가치를 깎음

### 대안 B: v1.36만 지원 (single version)
- 장점: 가장 단순, 최신 기능 풀 활용
- 단점: 사내 클러스터 채택률 낮음 (보통 1~2 버전 뒤)
- **기각 사유**: 도입 마찰이 너무 큼

### 대안 C: v1.32를 최소로
- 장점: 사내 보수 환경 추가 커버
- 단점:
  - K8s v1.32 patch support EOL은 **약 2026-02** (release 2024-12-11 + 14 months)이라 본 ADR 채택 시점(2026-05-18)에 이미 종료된 버전 → 보안 패치 없는 클러스터를 지원하는 셈
  - DRA(v1.34 GA), In-place Resize(v1.35 GA)도 미지원 → 핵심 차별화 약화
- **기각 사유**: 보안 정책 위반 가능성 + 차별화 약화

### 대안 D: 현재 결정 (v1.34 minimum, v1.36 target) — **채택**
- v1.34는 2026-05 시점에서 N-2 → 합리적 minimum
- 핵심 신규 기능(DRA, In-place Resize, Native Sidecar, Gateway API) 모두 GA
- 1년 후 자연스럽게 v1.36 minimum으로 상승 (rolling policy)

---

## 5. 후속 액션 (Follow-ups)

- [x] `docs/20-specs/k8s-integration-spec.md` 작성 (본 ADR과 페어) — 완료 2026-05-18
- [ ] `docs/40-runbooks/k8s-compat-matrix.md` 작성
- [ ] `go.mod`에 `k8s.io/client-go v0.36.x` 핀
- [ ] CI 매트릭스에 v1.34/35/36 kind/k3s 추가
- [x] `coroot-mapping.md`에 K8s 정책 반영 (병행) — 완료 2026-05-18 (§1.3.5)

---

## 6. 변경 이력

| 일자 | 작성자 | 비고 |
|---|---|---|
| 2026-05-18 | SubOrchestrator 초기 분석 | Draft 초안 |
