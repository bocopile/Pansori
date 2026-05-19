# ADR-0003-1: 본 프로젝트 SPDX 라이선스 — **Apache-2.0**

| 메타 | 값 |
|---|---|
| **상태** | Accepted |
| **결정일** | 2026-05-18 |
| **결정자** | 사용자 |
| **부모 ADR** | ADR-0003 (라이선스 정책 — 순수 영감) |
| **관련 문서** | `CLAUDE.md` §2, `LICENSE` (TBD 생성), `NOTICE` (TBD 생성) |

---

## 1. 컨텍스트 (Context)

ADR-0003에서 본 프로젝트의 SPDX 라이선스 결정은 본 후속 ADR에 위임되었다. ADR-0003은 "AGPL을 채택하지 않는다"는 점만 확정했고, 구체 라이선스는 사내 정책에 따라 선택하기로 했다.

검토된 후보:
- **Apache-2.0**: 엔터프라이즈 OSS 표준, 특허 그랜트 명시, 기여자 보호
- **MIT**: 가장 단순, 특허 조항 없음
- **Proprietary**: 사내 독점
- **BSL (Business Source License)**: 소스 공개되나 상용 제한

---

## 2. 결정 (Decision)

### 2.1 채택: **Apache License 2.0** (`SPDX-License-Identifier: Apache-2.0`)

### 2.2 채택 사유
1. **특허 그랜트 (가장 큰 이점)**: 기여자가 본 프로젝트에 기여한 코드에 대한 자신의 특허를 비명시적으로 라이선스. 사내·사외 기여자 모두 보호.
2. **엔터프라이즈 OSS 표준**: K8s / Prometheus / OpenTelemetry / etcd 등 본 프로젝트가 의존하는 핵심 생태계와 동일 라이선스 → 호환성 검증 부담 최소
3. **AGPL/GPL 의도적 회피**: Apache-2.0은 GPL 가족과 한 방향 비호환 → ADR-0003의 "AGPL 안 만짐" 정책을 라이선스 차원에서 추가 강화
4. **사외 배포 옵션 보존**: 향후 사외 배포·기여자 받기 결정 시 통과
5. **기여자 라이선스 명료**: CLA(Contributor License Agreement) 명시적 처리 가능

### 2.3 라이선스 호환 매트릭스 (의존성 선택 시 참고)

| 의존성 라이선스 | 본 프로젝트(Apache-2.0)와 호환? |
|---|---|
| Apache-2.0 / Apache-1.1 | ✅ 호환 |
| MIT / BSD (2/3-clause) / ISC | ✅ 호환 |
| MPL-2.0 / EPL-2.0 | ✅ 호환 (조건부 — 파일 단위 copyleft) |
| LGPL (v2.1/v3) | ⚠️ 검토 필요 — 동적 링크만 |
| GPL-2.0 (단독) | ❌ 비호환 |
| GPL-3.0 | ⚠️ 한 방향 호환 (Apache-2.0 → GPL-3.0 OK, 역방향 X) |
| **AGPL-3.0** | **❌ 비호환** — ADR-0003 정책과 일치 |
| Proprietary / Commercial | 케이스별 검토 |

### 2.4 강제 사항 (MUST)

#### 2.4.1 LICENSE 파일
- 저장소 루트에 `LICENSE` 파일 — Apache-2.0 전문 사본 (https://www.apache.org/licenses/LICENSE-2.0.txt)

#### 2.4.2 NOTICE 파일
- 저장소 루트에 `NOTICE` 파일 — 저작권 표기와 사용 OSS의 attribution
- 의존성 추가 시 NOTICE 갱신

#### 2.4.3 SPDX 헤더 (모든 신규 코드)
모든 신규 작성 `.go`, `.ts`, `.vue`, `.sh`, `.yaml` 파일 상단에:

```go
// SPDX-FileCopyrightText: 2026 <사내 조직명>
// SPDX-License-Identifier: Apache-2.0

package <name>
```

TypeScript / Vue:
```ts
// SPDX-FileCopyrightText: 2026 <사내 조직명>
// SPDX-License-Identifier: Apache-2.0
```

YAML / Shell:
```yaml
# SPDX-FileCopyrightText: 2026 <사내 조직명>
# SPDX-License-Identifier: Apache-2.0
```

#### 2.4.4 의존성 추가 시 검증
- `go-licenses report ./...` 결과에 §2.3의 ❌ 라이선스가 없어야 함
- ⚠️ 라이선스는 ADR 작성 후 도입 (특히 LGPL은 동적 링크 확인)

### 2.5 사내 조직명 / Copyright Holder
- `<사내 조직명>` placeholder는 사내 OSS 정책에 따라 결정 (예: 사명, 사업부, 또는 개인 ID)
- 현재 시점에는 `TBD` — 사내 OSS 사무국 또는 법무 가이드 따름. 일관성 위해 한 곳에 결정 (`.editorconfig` 또는 `docs/30-adr/0003-2-copyright.md` 후속 ADR에서)

---

## 3. 결과 (Consequences)

### 3.1 긍정적
- 호환성 검증 부담 최소화 (사실상 모든 인기 OSS와 호환)
- 특허 분쟁 위험 감소 (특허 그랜트 + 종료 조항)
- 사내·사외 기여자 동일 조건
- ADR-0003 정책과 라이선스 레벨 일관
- 사외 배포 결정 시 추가 작업 거의 없음

### 3.2 부정적 / 제약
- AGPL/GPL-2.0 코드는 의존성으로도 사용 불가 — `go-licenses` 등으로 차단 필수
- 일부 사내 정책이 모든 OSS를 차단하는 환경이면 별도 협의 필요 (이 경우 Proprietary로 변경 검토)

### 3.3 미해결 / 후속
- 사내 OSS 정책이 정식 OSS 출시를 막을 경우: 본 ADR을 supersede하는 ADR-0003-2 작성 (Apache-2.0 → Proprietary)
- 사내 조직명/copyright holder 확정 (별도 ADR 또는 사용자 결정)

---

## 4. 후속 액션 (Follow-ups)

- [ ] **`LICENSE` 파일 생성** (Apache-2.0 전문) — 루트에 — SubOrchestrator가 P1 시작 시 또는 즉시
- [ ] **`NOTICE` 파일 생성** — 저작권 + 사용 OSS attribution placeholder
- [ ] **CLAUDE.md §2 갱신**: "TBD" → "Apache-2.0"
- [ ] **모든 신규 .go/.ts/.vue/.yaml 파일에 SPDX 헤더 추가** (코딩 규약에 명문화)
- [ ] **`docs/coding-style.md` §4 또는 §5에 SPDX 헤더 표준 추가**
- [ ] **CI에 의존성 라이선스 스캔** (`go-licenses report`) — AGPL/GPL-2.0 의존성 발견 시 빌드 실패
- [ ] **`copyright holder` placeholder 확정** — 사내 표준 따라

---

## 5. 검토된 대안

| 대안 | 기각 사유 |
|---|---|
| **MIT** | 특허 조항 없음 → 기여자 보호 약함. K8s 생태계와 라이선스 다양성 증가 |
| **Proprietary** | 사외 배포 옵션 포기. 본 시점에 미리 닫을 필요 없음 |
| **BSL** | 상용 제한 — 사외 배포 시 추가 협상 필요. 운영 부담 큼 |
| **AGPL** | ADR-0003에 의해 원천 금지 |

---

## 6. 변경 이력

| 일자 | 변경 |
|---|---|
| 2026-05-18 | 초안 (사용자 결정: Apache-2.0 채택) |
