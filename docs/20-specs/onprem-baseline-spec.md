# SPEC: On-Premise 운영 베이스라인

| 메타 | 값 |
|---|---|
| **상태** | Draft v0.1 |
| **작성일** | 2026-05-18 |
| **오너** | TBD |
| **관련 ADR** | ADR-0001 (K8s 버전), ADR-0002 (기능 상태표) |
| **영향 모듈** | 거의 전체 — 특히 `modules/agent-*`, `deploy/*`, `pkg/*` |

---

## 1. 목적

On-Premise 환경(사내망 + 제한적 외부 게이트웨이)에서 본 시스템이 **보안·안정성·운영성** 기준선을 만족하도록 횡단 정책을 단일 문서로 정리한다.

3사 AI 검증(Codex / Claude / Gemini) 과정에서 누락이 식별된 항목을 모두 포함한다:
- PSA 프로파일 / NetworkPolicy 베이스라인
- 시간/타임존 / 사내 DNS
- 이미지 서명·SBOM / 공급망 보안
- eBPF 운영 가드레일 (map size, capability, bpffs)
- 메트릭 카디널리티 한도
- 백업/복구 / Storage retention & self-preservation
- 인증서 자동 갱신 / 사내 PKI
- Air-gapped 운영 절차
- Audit Log 정책
- proxy 표준 / OS 호환

---

## 2. 보안 (Security Baseline)

### 2.1 Pod Security Admission (PSA) 프로파일

| 네임스페이스 | 프로파일 | 사유 |
|---|---|---|
| `coroot-system` | `restricted` | 서버/UI/operator는 일반 워크로드 |
| `coroot-agents` | `privileged` | **eBPF 에이전트가 hostPID/hostNetwork/CAP_BPF/CAP_PERFMON 필요** |
| 사용자 워크로드 ns | (사내 정책 — 본 프로젝트 영역 아님) | — |

#### `coroot-agents` 네임스페이스의 정당화
- node-agent는 다음을 필요로 함:
  - `hostNetwork: true` (호스트 네트워크 가시성)
  - `hostPID: true` (프로세스 디스커버리)
  - `securityContext.capabilities: [CAP_BPF, CAP_PERFMON, CAP_NET_ADMIN, CAP_SYS_PTRACE]` (eBPF 부착, perf 이벤트)
  - `volumeMounts`: `/sys/fs/bpf` (bpffs), `/sys/kernel/tracing`, `/proc`, `/sys`
- 따라서 PSA `privileged`로 운영, 단 ns 자체를 **에이전트 전용으로 격리**

### 2.2 NetworkPolicy 베이스라인

```yaml
# coroot-system: server는 storage/IdP/LLM/Alarm만 outbound 허용
# coroot-agents: 에이전트는 coroot-server만 outbound 허용 (mTLS port)
# 모든 ns: ingress는 명시적 NetworkPolicy로만 (default-deny)
```

| 컴포넌트 | Egress 허용 | Ingress 허용 |
|---|---|---|
| coroot-server | Prom, CH, PG, SSO, LLM-GW(옵션), Alarm-GW | UI(443), agents(mTLS 4317/4318), operator |
| node-agent | coroot-server (4317/4318), K8s API | (없음 — pure client) |
| cluster-agent | coroot-server, K8s API | (없음) |
| operator | K8s API, OCI Registry (이미지) | (없음) |
| ClickHouse/Prom/PG | (사내 표준) | coroot-server만 |

### 2.3 SecurityContext 표준 (server/UI/operator)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532    # nonroot
  fsGroup: 65532
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
```

### 2.4 RBAC 최소 권한 원칙 (재확인 — `k8s-integration-spec.md` §6 단일 출처)

- ❌ `cluster-admin`, wildcard `*` resource, 전체 secret watch 금지
- ✅ agent-node: nodes/pods/endpointslices read만
- ✅ agent-cluster: 광범위 read, write 0
- ✅ operator: 자체 CRD RW + `pods/resize` (write), `admissionregistration.k8s.io/*` (write)

### 2.5 시크릿/인증

- **모든 시크릿은 K8s Secret**(KMS v2 etcd 암호화 권장) 또는 **사내 Vault Inject** 사용
- 환경변수에 평문 시크릿 노출 금지 — Vault Agent / Secrets Store CSI Driver 우선
- 시크릿 회전: 인증서는 24h(자동), DB 비밀번호는 90일(운영 정책)
- log redaction: `pkg/log/redact.go` 통해 자동 마스킹

---

## 3. PKI / 인증서 자동 갱신

### 3.1 인증서 체계

```
사내 Root CA
   └── 사내 Intermediate CA (Vault PKI / 사내 CA Operator)
         ├── coroot-server-ingress  (장기, 1년)
         ├── coroot-server-mtls     (24h, 자동 갱신)
         ├── agent-* (per agent)    (24h, 자동 갱신)
         └── operator-webhook       (1년)
```

### 3.2 자동 갱신 메커니즘

- **K8s 환경**: `cert-manager` + 사내 CA Issuer (또는 Vault Issuer)
- **VM 환경**: `vault agent` + systemd timer (12h 주기 갱신 시도)
- **에이전트 부트스트랩**: 단기 부트스트랩 토큰 → 본 인증서 CSR → 사내 CA 승인 → 발급

### 3.3 CSR 자동 승인 정책
- agent-cluster가 부트스트랩 토큰을 검증한 노드만 CSR을 사내 CA에 forward
- 사내 CA는 사전 등록된 노드 인벤토리(또는 SPIFFE) 기준으로 자동 승인

---

## 4. 시간 / NTP / 타임존

| 항목 | 표준 |
|---|---|
| **노드 시간 동기화** | NTP/Chrony 필수. 사내 NTP 서버(`ntp.<사내>.kr`) 기본. drift 1초 초과 시 alert |
| **타임존 — 운영 저장** | **UTC** (Prometheus, ClickHouse, PG 내부 timestamp) |
| **타임존 — UI 표시** | 기본 `Asia/Seoul` (사용자 프로파일에서 변경 가능) |
| **타임존 — 로그** | RFC3339 with offset (`2026-05-18T15:30:00+09:00`) |
| **시계 정확도 의존 기능** | 시계 drift 1초 초과 시 SLO 계산/이상 탐지 일시 중단 |

---

## 5. DNS / Proxy / 네트워크

### 5.1 DNS 정책

- **클러스터 내부**: kube-dns/CoreDNS 표준
- **사내 도메인 분기**: CoreDNS forward 룰로 `<사내>.kr` → 사내 DNS
- **외부 DNS**: 평시 차단. 부트스트랩 시에만 사내 미러 도메인 화이트리스트

### 5.2 Proxy 환경변수 표준

```bash
HTTP_PROXY=http://proxy.<사내>.kr:8080
HTTPS_PROXY=http://proxy.<사내>.kr:8080
NO_PROXY=cluster.local,.svc,.svc.cluster.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.<사내>.kr,<Pod CIDR>,<Service CIDR>,localhost,127.0.0.1
```

- `pkg/httpx/Client`는 위 환경변수를 반드시 존중
- **NO_PROXY에 Pod CIDR / Service CIDR / `cluster.local`** 누락 시 클러스터 내부 트래픽이 proxy 경유 → 장애. 운영 표준화 필수

### 5.3 K8s 외부 의존 도메인 화이트리스트 (예시)

| 용도 | 도메인 | 비고 |
|---|---|---|
| SSO | `idp.<사내>.kr` | OIDC/SAML |
| OCI Registry | `registry.<사내>.kr` | 이미지/Helm 미러 |
| LLM Gateway (옵션) | `llm-gw.<사내>.kr` | RCA |
| Alarm GW | `alarm.<사내>.kr` | Slack/메일 발송 |
| NTP | `ntp.<사내>.kr` | |
| Vault | `vault.<사내>.kr` | PKI/시크릿 |

평시 외부 인터넷 접근은 모두 차단.

> ⚠️ **FQDN 기반 egress의 전제**: 표준 K8s `networking.k8s.io/v1 NetworkPolicy`는 **FQDN 매칭을 직접 지원하지 않는다** (IP/CIDR/podSelector/namespaceSelector만 지원). FQDN 화이트리스트는 다음 중 하나의 CNI 확장이 전제된다:
> - **Cilium**: `CiliumNetworkPolicy` + `toFQDNs` 규칙 (1순위 권장 — 이미 eBPF 스택과 시너지)
> - **Calico**: `GlobalNetworkPolicy` + DNS-based selector
> - **외부 egress gateway**: 사내 forward proxy + IP allowlist
>
> CNI 미설치 환경에서는 **노드 OS의 firewalld/nftables** 또는 사내 proxy 라우팅으로 동등 제어. CNI 선택은 `docs/40-runbooks/cni-policy.md` (TBD)에서 별도 결정.

---

## 6. 공급망 보안 (Supply Chain)

### 6.1 이미지 / SBOM / 서명

- **빌드**: 사내 CI에서만 빌드. 외부 GitHub Actions 등 미사용
- **SBOM**: `syft` 또는 사내 표준으로 SPDX/CycloneDX 발행, 사내 SBOM 저장소 업로드
- **서명**: `cosign` 으로 사내 키 서명. 모든 이미지에 `cosign verify` 통과 필수
- **취약점 스캔**: `trivy` 또는 `grype`로 빌드 시 차단 (Critical/High 발견 시 빌드 실패)
- **태그 정책**: `:latest` 금지. semver 또는 git sha 기반

### 6.2 K8s 측 검증

다음은 **단일 admission 메커니즘만으로는 불가능**하다. 각 룰을 적절한 도구에 매핑해 적용한다:

| 검증 항목 | 적용 도구 | 비고 |
|---|---|---|
| 이미지가 사내 Registry에서 왔는가 | **ValidatingAdmissionPolicy + CEL** | CEL이 `image` 필드 prefix만 검사하면 충분 — VAP 단독 가능 |
| cosign 서명이 유효한가 | **`policy-controller` (sigstore)** 또는 **Kyverno** | CEL은 외부 서명 검증 불가 — 별도 admission 컨트롤러 필수 |
| SBOM이 SBOM 저장소에 존재하는가 | **policy-controller / Kyverno + 사내 SBOM 저장소 API** | 외부 데이터 조회 필요 — CEL 단독 불가 |
| Pod이 `coroot-agents` ns 외에서 privileged인가 | **ValidatingAdmissionPolicy + CEL** | PSA enforce와 함께 |

> ⚠️ **PSA `coroot-agents=privileged` exemption**: agent Pod은 위 cosign/SBOM 룰을 통과하되 privileged 제한에서는 제외되도록, 룰셋에 명시적 `excluded-namespaces` 또는 `matchExpressions` 추가 필요.

### 6.3 Helm/매니페스트 검증
- `helm template ... | kubeval` 또는 `kubectl-validate` 실행
- `pluto` 로 deprecated API 차단
- Helm chart digest pinning 필수, `helm dependency update` 결과 lock 커밋

---

## 7. eBPF 운영 가드레일

### 7.1 커널/런타임 요건

| 항목 | 요건 |
|---|---|
| **커널** | 5.10+ (BTF 활성). 일부 helper는 5.16+. **BTF/helper capability probe**로 동적 확인 |
| **컨테이너 런타임** | `containerd 1.7+` (Docker 미지원) 또는 CRI-O (검토 후 결정) |
| **bpffs** | `/sys/fs/bpf` mount 필수 (hostPath) |
| **SELinux/AppArmor** | 모드 enforce에서도 동작하도록 정책 라벨/프로파일 동봉 (`policies/`) |

### 7.2 OS 호환 매트릭스 (1차)

> **정책 (CLAUDE.md §2.5 / ADR-0001 §3.3과 일치)**: 커널 ≥ 5.10 + BTF 활성을 **hard requirement**로 한다. 미달 노드는 **미지원**.

| OS | 커널 | 지원 |
|---|---|---|
| RHEL 9 / Rocky 9 / Alma 9 | 5.14+ | ✅ |
| Ubuntu 22.04 LTS | 5.15+ | ✅ |
| Ubuntu 24.04 LTS | 6.8+ | ✅ |
| Amazon Linux 2023 | 6.1+ | ✅ |
| SUSE SLES 15 SP5+ | 5.14+ | ⚠️ 검증 필요 (HW/CNI 호환만 추가 확인) |

#### 명시적 비지원

| OS | 커널 | 사유 |
|---|---|---|
| RHEL 8 / Rocky 8 / Alma 8 | 4.18 | eBPF helper/CO-RE 호환 부족, fentry/fexit 미지원 → 핵심 기능 결손 |
| CentOS 7 / RHEL 7 | 3.10 | 미지원 (EOL OS) |

> ⚠️ RHEL 8 지원이 사업적으로 필요한 경우: **별도 ADR로 도입 검토**. degraded mode(kprobe only, profile 비활성) 정의가 별도 작업으로 필요하며, 본 SPEC의 현재 정책에는 포함되지 않는다.

### 7.3 자원 가드레일 (per node-agent)

| 한도 | 기본값 | 동작 |
|---|---|---|
| `eBPFMapMaxEntries` (per map) | 100,000 | 초과 시 LRU eviction + warn |
| `eBPFMapsTotal` | 32 | 초과 시 신규 attach 거부 |
| 메모리 한도 | 512 MiB | OOM 직전 90%에서 자체 GC |
| CPU 한도 | 200 mCPU | 초과 시 sampling 자동 강화 |
| L7 parser 동시 연결 | 50,000 | 초과 시 신규 연결 미관측 |

### 7.4 안전장치

- **Pre-flight check**: 커널 버전, BTF 존재, 필요 helper 존재, bpffs mount 확인 → 미달 시 명확한 에러 + 자체 비활성
- **panic 방지**: 모든 eBPF 부착 실패는 graceful degrade (해당 tracer만 비활성, 다른 tracer 계속)
- **고정밀 perf event는 옵션**: 노드 부담 보고 권장 비활성

---

## 8. 메트릭 / 로그 / 트레이스 운영

### 8.1 메트릭 카디널리티 한도

| 항목 | 한도 | 동작 |
|---|---|---|
| 라벨당 unique 값 | 10,000 | 초과 라벨은 `__too_many_values__`로 압축 + alert |
| 메트릭당 시리즈 수 | 100,000 | 초과 시 신규 시리즈 거부 + alert |
| 사용자 정의 라벨 | 사전 정의된 키만 (`pkg/labels/allowlist.go`) | allowlist 외 라벨 자동 드롭 |
| 한글 라벨 값 (Prom UTF-8 허용) | OK이나 길이 ≤ 64자 | 길이 초과 시 truncate |

### 8.2 데이터 보존 (Retention) — 기본값

| 신호 | 저장소 | 기본 보존 | 옵션 |
|---|---|---|---|
| 메트릭 (high cardinality) | Prometheus | 30일 | 30/60/90일 |
| 메트릭 (집계) | Prometheus / Thanos (옵션) | 1년 | 사내 정책 |
| 트레이스 | ClickHouse | 7일 | 1/7/14일 |
| 로그 | ClickHouse | 14일 | 7/14/30일 |
| 프로파일 | ClickHouse | 3일 | 1/3/7일 |
| 이벤트 (K8s events) | ClickHouse | 30일 | |
| 감사 로그 (audit) | 별도 ClickHouse 테이블 | **365일** | 사내 보안 정책 |

### 8.3 Self-Preservation (디스크 풀 방지)

```
디스크 사용량 80% → INFO 알람
디스크 사용량 85% → 수집 제한 모드 (sampling 강화, 카디널리티 한도 절반)
디스크 사용량 90% → 신규 수집 거부 시작 (에이전트로 backoff 신호)
디스크 사용량 95% → 즉시 수신 중단 + 운영자 emergency alert
```

- 모든 수신기는 backpressure 시그널을 OTLP/PromRW 응답으로 에이전트에 전달
- 에이전트는 backpressure 수신 시 로컬 큐 폐기 (오래된 것부터)

---

## 9. 백업 / 복구

| 컴포넌트 | 백업 주기 | 보관 | 복구 검증 |
|---|---|---|---|
| **PostgreSQL (메타)** | **일 1회 + WAL 연속** | 30일 | 월 1회 dry-run |
| Prometheus 메트릭 | 사내 표준 (옵션) | — | — |
| ClickHouse | parts 백업 옵션 (S3 호환 사내 오브젝트 스토리지) | 7일 | 분기 1회 dry-run |
| Operator state | git 형상 + K8s manifest 이미 형상관리 | — | — |
| 사내 PKI 발급 인증서 | Vault 자체 백업 | — | — |

**RPO/RTO**:
- 메타 DB: RPO 1h / RTO 4h
- 텔레메트리: best-effort (텔레메트리 자체 손실은 SLO 외)

---

## 10. Audit Log 정책

### 10.1 K8s Audit Log

> ⚠️ **수집 경로 정정**: K8s audit log는 **kube-apiserver의 audit policy + backend(file 또는 webhook)** 에서 노출된다. **agent의 K8s API watch로 가져올 수 없다**.

지원 수집 경로 (택일):

| 경로 | 설명 | 권장 환경 |
|---|---|---|
| **Webhook backend** | kube-apiserver `--audit-webhook-config-file`로 audit-webhook을 cluster-agent의 audit endpoint로 forward | kubeadm/Kubespray 등 apiserver flag 제어 가능 |
| **File backend + sidecar 수집** | kube-apiserver `--audit-log-path`로 파일 출력 → 별도 log shipper(fluent-bit 등)가 ClickHouse로 전송 | managed K8s 또는 file 정책이 표준인 사내 |
| **수집 비활성** | audit log를 수집하지 않음 | 사내 보안팀이 별도 audit 파이프라인 운영 중일 때 |

- ClickHouse `k8s_audit` 테이블에 365일 보존
- 보안팀 UI 뷰로 분리
- cluster-agent는 webhook 수신 endpoint를 노출하는 모드를 옵션 제공

### 10.2 본 시스템 자체의 감사 로그
- 사용자 액션 (로그인/RBAC 변경/SLO 룰 변경/알람 채널 변경) = 감사 이벤트
- ClickHouse `app_audit` 테이블, 365일 보존
- 절대 삭제 불가 (append-only)

---

## 11. Air-gapped 운영 절차

### 11.1 준비물
- 사내 OCI Registry (Harbor / Nexus 등)
- 사내 Helm Repo (또는 OCI 기반 Helm)
- 사내 Go 모듈 프록시 (Athens / GOPROXY)
- 사내 NPM 미러
- 사내 SBOM 저장소

### 11.2 부트스트랩 절차
1. 외부에서 (한 번) 이미지·차트·모듈을 동기화 → 사내 Registry에 미러
2. 사내 환경에서 `helm install --set image.registry=registry.<사내>.kr ...`
3. 모든 외부 도메인 호출이 차단된 상태에서 동작 확인 (`netpol-test`)

### 11.3 업그레이드 절차
- 분기별 사내 동기화 윈도우(예: 매월 2주차 목요일) 동기화
- 사내 staging cluster에서 검증 → 운영 클러스터 롤아웃

### 11.4 금지 사항
- ❌ Helm dependency를 외부 chart 저장소 직접 참조 (모두 vendoring or 사내 미러)
- ❌ `go get` 시점에 외부 GOPROXY 직접 호출
- ❌ npm install 시 외부 registry 사용

---

## 12. OS / 노드 운영

### 12.1 systemd unit 표준 (VM 노드용)

```ini
# /etc/systemd/system/coroot-node-agent.service
[Unit]
Description=Coroot Node Agent (eBPF telemetry)
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=root
EnvironmentFile=/etc/coroot/agent.env
ExecStartPre=/usr/bin/coroot-agent preflight
ExecStart=/usr/bin/coroot-agent run --config /etc/coroot/agent.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
LimitNOFILE=1048576
LimitMEMLOCK=infinity
AmbientCapabilities=CAP_BPF CAP_PERFMON CAP_NET_ADMIN CAP_SYS_PTRACE
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/coroot /sys/fs/bpf

[Install]
WantedBy=multi-user.target
```

### 12.2 패키지 배포
- RPM (RHEL/Rocky/SLES) + DEB (Ubuntu) 동시 배포
- 사내 yum/apt 미러 등록

---

## 13. 미해결 사항 (TBD)

- [ ] Windows 노드 지원 로드맵 (eBPF 대신 ETW + WinSpan?) — P6+ 별도 ADR
- [ ] OpenShift 4.18+ 호환 (SCC vs PSA)
- [ ] CRI-O 지원 여부
- [ ] 사내 IdP가 SAML 전용일 때 OIDC 어댑터 필요 여부
- [ ] 사내 Vault 미보유 환경에서의 시크릿 표준 (K8s Secret + KMS v2만으로 충분?)
- [ ] 멀티 클러스터 페더레이션 (관측 대상 다중 클러스터)
- [ ] 데이터 분류 (PII/민감/일반) 정책 — `pkg/log/redact.go` 룰 정의

---

## 14. 체크리스트 (Definition of Done per 컴포넌트)

각 모듈은 다음을 만족해야 머지:

- [ ] PSA 프로파일 적용 (Pod manifest 검증)
- [ ] NetworkPolicy 동봉
- [ ] securityContext 표준 적용 (또는 명시적 예외 사유)
- [ ] proxy 환경변수 존중 (`pkg/httpx` 사용 확인)
- [ ] 모든 외부 호출 timeout/retry/circuit-breaker
- [ ] 메트릭 카디널리티 한도 통과
- [ ] 로그 redaction 적용
- [ ] SBOM/서명 빌드 파이프라인 통과
- [ ] Pre-flight check 구현 (해당 시)
- [ ] Helm chart `kubeVersion: ">=1.34.0-0"` 명시
- [ ] 본 SPEC의 미해결 사항에 본 모듈이 추가 항목을 만들지 않는지 확인

---

## 15. 변경 이력

| 일자 | 변경 |
|---|---|
| 2026-05-18 | 초안 (3사 AI 교차검증에서 식별된 누락 항목 통합) |
