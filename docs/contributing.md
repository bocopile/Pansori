# Contributing — 의존성 / 커밋 / PR / 문서 거버넌스

> 본 문서는 본 프로젝트의 **공정 규약**(의존성 추가·커밋·PR·문서 변경) 단일 출처다.
> 루트 `CLAUDE.md`는 본 문서를 링크로만 참조한다.

---

## 1. 의존성 추가 / 제거

### 1.1 라이선스 정책 (MUST)
본 프로젝트 SPDX = Apache-2.0 (ADR-0003-1). 의존성 라이선스는 **ADR-0003-1 §2.3 호환 매트릭스**를 따른다.

| 라이선스 | 허용 여부 |
|---|---|
| Apache-2.0 / MIT / BSD-2/3-clause / ISC / MPL-2.0 / EPL-2.0 | ✅ 허용 |
| LGPL (v2.1/v3) | ⚠️ 동적 링크만 허용 — ADR 필수 |
| GPL-2.0 / GPL-3.0 | ❌ 금지 (Apache-2.0과 비호환) |
| **AGPL (모든 버전)** | **❌ 원천 금지 — ADR-0003 §2.3에 의해. 별도 검토 대상이 아니라 즉시 거부** |
| Proprietary | 케이스별, 라이선스 비용·재배포 가능성 검토 |

### 1.2 새 외부 의존성 추가 (MUST) — 절차
1. **라이선스 확인**: §1.1 매트릭스 통과 필수. AGPL 발견 시 도입 시도 자체 중단
2. **유지보수 상태**: 최근 6개월 내 커밋, 명시적 v1.0+ 권장
3. **공급망 보안**: 알려진 CVE 확인 (`govulncheck` / npm audit)
4. **ADR 작성**: `docs/30-adr/NNNN-adopt-<package>.md` (단, `ADR-0002 §4` 또는 `contracts-spec.md §3` 등 기존 표에 등재된 라이브러리는 재 ADR 불요)
5. **Vendor 검토**: 작은 라이브러리는 vendor 고려

### 1.3 라이선스 스캔 게이트 (CI 강제)
P1 첫 PR 전 다음 스캔 도구 통합:

| 대상 | 도구 | 차단 기준 |
|---|---|---|
| Go 의존성 | `go-licenses report ./...` | AGPL / GPL-2.0 발견 시 빌드 실패 |
| npm/devDependency (UI, 빌드 도구) | `license-checker --excludePrivatePackages --failOn 'AGPL-3.0;GPL-2.0;GPL-3.0'` | 동일 |
| 컨테이너 base image | `syft` SBOM + `grype` / 사내 컴플라이언스 도구 | AGPL 라이브러리 포함 시 빌드 실패 |
| `oapi-codegen` / `buf` 등 빌드 도구 자체 | 위와 동일 (CI 환경 검증) | 동일 |

### 1.4 의존성 제거 (MUST)
- 마이그레이션 경로 명시 (대체 라이브러리 또는 자체 구현)
- 영향받는 모듈 전체 회귀 테스트

### 1.5 사내 패키지 우선 원칙 (SHOULD)
사내에 동등 라이브러리가 있으면 외부 OSS보다 우선 사용. 사내 패키지 존재 여부 확인 경로:
- 사내 패키지 인덱스 (TBD — 사내 표준 페이지가 결정되면 본 문서 갱신)
- 결정 전까지는 외부 OSS 검토 시 동시에 사용자에게 사내 등가물 여부 질문

---

## 2. 브랜치 / 커밋 / PR

### 2.1 브랜치 전략
- `main`: 보호. 직접 푸시 금지
- 작업 브랜치: `feat/<scope>-<short-desc>` / `fix/<scope>-<short-desc>`
- 예: `feat/agent-node-tcp-tracer`, `fix/storage-clickhouse-timeout`

### 2.2 커밋 메시지 (Conventional Commits)
```
<type>(<scope>): <subject>

<body>

<footer>
```
- type: `feat | fix | refactor | docs | test | chore | perf | build`
- scope: 모듈명 (`agent-node`, `storage`, `ui` 등)
- 한국어 본문 OK, 제목은 영문 권장

### 2.3 PR 규약
- 제목: 커밋 메시지와 동일 양식
- 본문: **변경 이유** + **검증 방법** + **위험요소** 필수
- 리뷰어: 최소 1명 (모듈 오너 + 1)
- CI 통과 필수: `build` / `test` / `lint` / `vulncheck`
- 매뉴얼 테스트 결과 또는 스크린샷 (UI 변경 시)

---

## 3. 작업 완료 정의 (Definition of Done)

- [ ] **검증 명령(`CLAUDE.md` §6 — SSOT) 원샷 통과**: `go build ./... && go test ./... && go vet ./... && golangci-lint run && govulncheck ./...`
- [ ] 단위 테스트 커버리지 ≥ 60% (핵심 로직 ≥ 80%)
- [ ] 공개 API에 GoDoc 주석
- [ ] HARD CONSTRAINTS (CLAUDE.md §3) 위반 없음 — self-check 필수
- [ ] 자명하지 않은 결정은 ADR 추가
- [ ] 영향받는 SPEC/ADR/매핑표 동시 갱신

---

## 4. 문서 변경 거버넌스

### 4.1 문서 우선순위
**SSOT: `CLAUDE.md` §8 (7단계 우선순위)** — 본 절은 링크만 보유.

### 4.2 CLAUDE.md 변경 규칙
- **PR 필수** + **모듈 오너 전원 리뷰**
- 변경 사유는 ADR로 기록 (`docs/30-adr/NNNN-claude-md-*.md`)
- 의미적 버전 라벨 사용 (문서 상단에 `v0.x` 명시)

### 4.3 ADR 작성 규칙
- 파일명: `docs/30-adr/NNNN-<kebab-case-title>.md` (NNNN은 4자리 번호)
- 섹션: 메타 / Context / Decision / Consequences / Alternatives / Follow-ups
- 새 결정은 기존 ADR을 supersede 가능 (`Supersedes: ADR-NNNN`)

### 4.4 SPEC 작성 규칙
- 파일명: `docs/20-specs/<kebab-case-name>-spec.md`
- 관련 ADR 명시
- TBD/Open Questions 섹션 필수
