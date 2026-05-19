# 글로서리 (도메인 용어)

> 본 프로젝트에서 사용하는 옵저버빌리티/K8s/eBPF 도메인 용어 정의.

---

## 옵저버빌리티

| 용어 | 정의 |
|---|---|
| **Service Map** | 서비스 간 통신 토폴로지 시각화 |
| **Inspection** | 특정 서비스의 건강 상태 점검 항목 (에러율, 지연 등) |
| **SLO** | Service Level Objective. 목표 가용성/성능 지표 |
| **SLI** | Service Level Indicator. 실제 측정값 (SLO와 비교 대상) |
| **RCA** | Root Cause Analysis. 장애 근본 원인 분석 |
| **RED 메트릭** | Rate / Errors / Duration — 요청 중심 |
| **USE 메트릭** | Utilization / Saturation / Errors — 자원 중심 |
| **카디널리티** | 메트릭 라벨의 unique 조합 수. 폭주 시 저장소 부담 |
| **Self-Observability** | 본 시스템 자체를 본 시스템으로 관측 |

---

## OpenTelemetry / 표준 프로토콜

| 용어 | 정의 |
|---|---|
| **OTLP** | OpenTelemetry Protocol. 표준 텔레메트리 전송 |
| **PromRW** | Prometheus Remote Write |
| **OTel** | OpenTelemetry 약칭 |
| **Profile signal** | OTel의 4번째 시그널 (alpha — 본 프로젝트 보류) |

---

## eBPF / 커널

| 용어 | 정의 |
|---|---|
| **eBPF** | Extended Berkeley Packet Filter. 커널 수준 안전 코드 실행 |
| **CO-RE** | Compile Once - Run Everywhere. 커널 헤더 비의존 빌드 |
| **BTF** | BPF Type Format. 커널 타입 정보 메타데이터 |
| **fentry/fexit** | eBPF 프로그램 부착 방식. 5.11+ 함수 진입/종료 |
| **kprobe** | 커널 동적 프로빙. fentry fallback |
| **bpffs** | `/sys/fs/bpf` BPF 객체 파일시스템 |

---

## 컴포넌트 / 모듈

| 용어 | 정의 |
|---|---|
| **agent-node** | 노드(호스트)당 1개 배포되는 eBPF 수집기 (디렉터리: `modules/agent-node/`) |
| **agent-cluster** | 클러스터/도메인당 1개 배포되는 광역 수집기 (디렉터리: `modules/agent-cluster/`) |
| **agent-db-\*** | DB(Postgres/MySQL 등)별 메트릭 수집기 (옵션, 디렉터리: `modules/agent-db-postgres/` 등) |
| **coroot-server** | API + 분석 엔진 + UI 임베드의 메인 백엔드 (디렉터리: `modules/api/` + `modules/processing/`) |
| **operator** | K8s CRD 기반 자동 배포/관리 컴포넌트 (디렉터리: `modules/operator/`) |
| **auth-svc** | 사내 SSO 연동 (server 임베드 또는 분리 모드, 디렉터리: `modules/auth/`) |

---

## Kubernetes 신규 기능 (v1.34+)

| 용어 | 정의 |
|---|---|
| **Native Sidecar** | initContainer + restartPolicy=Always (v1.33 GA) |
| **DRA** | Dynamic Resource Allocation. GPU/가속기 자원 추상화 (v1.34 GA) |
| **In-place Pod Resize** | 재시작 없이 Pod 리소스 조정 (v1.35 GA) |
| **Gateway API** | Ingress 차세대. L7 라우팅 (2023-10 GA) |
| **VAP** | ValidatingAdmissionPolicy. CEL 기반 admission (v1.30 GA) |
| **EndpointSlice** | Endpoints의 분산 친화 후속 (v1.21 GA) |
| **PSA** | Pod Security Admission. PSP 후속 (v1.25 GA) |

---

## 본 프로젝트 약어

| 용어 | 정의 |
|---|---|
| **SSOT** | Single Source of Truth. 단일 출처 |
| **ADR** | Architecture Decision Record |
| **PSA 프로파일** | `restricted` / `baseline` / `privileged` 중 하나 |
| **T1/T2/T3** | 기능 도입 우선순위. 필수/옵션-capability-gated/보류 |
