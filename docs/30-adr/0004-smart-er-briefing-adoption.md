# ADR-0004: SMART+ER 메타 프롬프트 헤더 채택 (CLAUDE.md §0)

| 메타 | 값 |
|---|---|
| **상태** | Accepted |
| **결정일** | 2026-05-20 |
| **결정자** | 사용자 (P0 단계 의사결정) |
| **관련 ADR** | ADR-0003 (라이선스 정책 — 순수 영감), ADR-0003-1 (SPDX = Apache-2.0) |
| **관련 문서** | `CLAUDE.md` §0 (본 ADR로 신설), `CLAUDE.md` §1, §10 (변경 규칙) |
| **외부 참조** | [modu-ai/smarter-prompt](https://github.com/modu-ai/smarter-prompt) (MIT, "Copyleft" 표기 비정형 — **아이디어만 차용, 코드/문서 직접 복사 없음**) |

---

## 1. 컨텍스트 (Context)

본 프로젝트는 다수의 AI 에이전트(Claude / Codex / Gemini CLI / SubOrchestrator 하위 에이전트)가 동시에 작업하는 멀티에이전트 거버넌스 구조다. 루트 `CLAUDE.md`는 모든 에이전트가 가장 먼저 읽는 "메타 프롬프트"로 작동한다.

3사(Claude / Codex / Gemini) CLI를 통한 교차검증 결과 ([기록: 본 ADR §6 출처]), `CLAUDE.md` v1.1 (Slim Router, 254줄)을 SMART+ER 프레임워크 기준으로 평가했을 때 다음과 같은 불균형이 식별되었다.

| 요소 | 평가 | 근거 |
|---|---|---|
| S (Situation) | ✅ 모범 | §2 정체성 / §1.0 P0 상태 / §3 제약 — 풍부 |
| A (Action Steps) | ✅ 모범 | §1 4단계 라우터 + 확인 게이트 |
| R (Resource) | ✅ 모범 | §8 7단계 우선순위 / §9 14개 링크 |
| M (Mission) | ⚠️ 부분 | 비전(§2)은 있으나 P0 종료 정량 기준 부재 |
| R (Result) | ⚠️ 부분 | DoD(§1.2)는 있으나 응답 포맷 통일 부재 |
| T (Tone & Style) | ❌ 미충족 | AI 응답 톤·길이·언어·금지어 미명시 |
| E (Example) | ❌ 미충족 | 안티패턴 14건 / 권장 골든 패턴 0건 |

3사 평균 점수 8.03 / 10. 약점이 `T·E·M`에 집중되어 신규 에이전트 진입 시 "어떤 톤으로·언제 끝났다고 보고할지" 자기판단이 흔들리는 문제 발생.

---

## 2. 결정 (Decision)

### 2.1 SMART+ER 7요소 메타 프롬프트 헤더를 `CLAUDE.md` §0로 채택한다.

7요소:

| 요소 | 정의 |
|---|---|
| **S** Situation | 에이전트가 이해해야 할 현재 상황·배경·페르소나 |
| **M** Mission | 달성해야 할 측정 가능한 목표 (수치·기한·검수자) |
| **A** Action Steps | 수행할 단계·순서·확인 게이트 |
| **R** Result | 결과물 형식·완료 기준 (응답 포맷 포함) |
| **T** Tone & Style | 응답 톤·언어·금지어·강도 |
| **E** Example (선택) | 권장 패턴·골든 예시 |
| **R** Resource (선택) | 참조 자료·문서 우선순위 |

### 2.2 채택 형식

- 위치: `CLAUDE.md` 상단 박스(파일 위치/약속/용어) 다음, `## 1. ⚡ Agent Quickstart` 직전.
- 길이: 약 12줄 (현재 `CLAUDE.md` §0 v1).
- 미확정 값은 `[TBD]` 마커로 표기. 본 ADR §3에서 점진 확정.

### 2.3 부수 변경

- `CLAUDE.md` L4의 길이 약속 `약 220줄 이하 (±10줄)` → `약 270줄 이하 (±10줄)`로 갱신. 현재 v1.1이 이미 254줄로 약속을 위반한 상태였고, §0 추가 후 266줄이 되므로 현실에 맞춰 정직하게 갱신.

---

## 3. P0 종료 정량 기준 (TBD 항목 — 본 ADR에서 점진 확정)

`CLAUDE.md` §0 M(Mission)의 `[TBD]` 값은 다음 항목으로 본 ADR에서 후속 결정한다. 현재 결정 시점에서는 공란.

| 항목 | 현재값 | 확정 책임 | 확정 시점 |
|---|---|---|---|
| P0 종료 ADR 최소 건수 | `[TBD]` | 모듈 오너 합의 | 본 ADR 차기 개정 |
| P0 종료 SPEC 최소 건수 | `[TBD]` | 모듈 오너 합의 | 본 ADR 차기 개정 |
| P0 종료 기한 | `[TBD]` | 사용자 | 본 ADR 차기 개정 |
| `docs/templates/{adr,spec,contracts}.md` 경로 | `[TBD: P1에서 생성]` | SubOrchestrator dispatcher | P1 진입 시 |

본 ADR은 위 값들이 확정될 때마다 개정(amend)되며, 개정 시 `CLAUDE.md` §0의 해당 `[TBD]`도 동시 갱신한다.

---

## 4. 대안 (Alternatives Considered)

| 대안 | 평가 | 기각 사유 |
|---|---|---|
| (A) ABCDE 프레임워크 (Anthropic 공식) | 영문 / 5요소 | 한국어 도메인 적합도 낮음, E(Example) 강조 약함 |
| (B) RTF (Role-Task-Format) | 3요소, 간결 | A(Action Steps) 부재 — 본 프로젝트의 라우터 구조와 부합 안 함 |
| (C) CRISPE | 5요소 + Persona | T(Tone) 약함 |
| (D) 자체 프레임워크 신설 | 100% 맞춤 | 외부 표준 부재 → 신규 에이전트 학습 비용 증가 |
| **(E) SMART+ER (채택)** | 한국어 출처, 7요소, E/R 선택 | 본 프로젝트의 약점(T·E·M)을 모두 커버, 한국어 1차 출처 존재 |

---

## 5. 결과 (Consequences)

### 5.1 양성 영향
- 신규 에이전트(추후 도입될 LLM 포함)의 컨텍스트 흡수 비용 감소.
- AI 응답 톤/형식 통일 → 사용자 검토 비용 감소.
- SMART+ER E(Example) 슬롯 확보 → P1 진입 시 `docs/templates/` 작성 동기 부여.

### 5.2 부담 / 트레이드오프
- `CLAUDE.md` 12줄 증가 (266줄). 길이 약속 갱신 필요 (§2.3).
- P0 종료 정량 기준(`[TBD]`)이 본 ADR 차기 개정까지 미확정 → 에이전트는 "P0 종료 시점은 사용자 결정 사항"으로 인지해야 함.

### 5.3 후속 작업
- (a) `docs/templates/{adr,spec,contracts}.md` 생성 — P1 진입 시 (담당: SubOrchestrator dispatcher).
- (b) 본 ADR §3의 P0 종료 정량 기준 확정 — 모듈 오너 합의 회의.
- (c) `CLAUDE.md` §1.4 확인 게이트에 "ADR 신규=사용자 확인必"이 명시되어 있으므로 본 ADR은 사용자 승인 후에만 `Accepted`로 전환.

---

## 6. 출처

- 3사 CLI 교차검증 raw 로그: `/tmp/cli-eval-sp/{codex2,gemini2,claude2,codex3,gemini3,claude3}.log` (본 ADR 작성 시점 보관)
- SMART+ER 1차 출처: [modu-ai/smarter-prompt — README.md](https://github.com/modu-ai/smarter-prompt/blob/main/README.md)
- 본 프로젝트 라이선스 정책: ADR-0003
