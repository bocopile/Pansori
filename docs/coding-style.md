# 코딩 스타일 가이드

> 본 문서는 본 프로젝트 전체에 적용되는 **코딩 컨벤션**의 단일 출처(SSOT)다.
> 루트 `CLAUDE.md`는 본 문서를 링크로만 참조한다.

---

## 1. Go

### 1.1 포맷/린트 (자동화)
- **포맷**: `gofmt` + `goimports` 강제 (commit hook으로 자동 적용)
- **린터**: `golangci-lint` (설정 `.golangci.yml`)
- **CI**: `go vet` + `golangci-lint` + `govulncheck` 모두 통과 필수

### 1.2 에러 처리 (MUST)
- `fmt.Errorf("...: %w", err)`로 wrap — 원인 사슬 유지
- `_ = err` 무시 금지. 의도적이면 `// nolint:errcheck` + 사유 주석
- 사용자/UI 노출 에러는 `pkg/errs/` 경유 (라벨링·국제화)

### 1.3 컨텍스트 / 동시성
- 모든 외부 호출/IO 함수는 `ctx context.Context`를 **첫 인자**로
- `goroutine`은 반드시 종료 조건 명시 (ctx 또는 done 채널)
- `panic`은 라이브러리/서비스 코드에서 금지. 초기화 실패만 허용

### 1.4 타입/디자인
- **인터페이스는 작게** (1~3개 메서드). 큰 인터페이스는 컴포지션
- `interface{}` / `any` 남발 금지. 제네릭(Go 1.21+) 또는 구체 타입 우선
- **글로벌 변수 금지** — 예외: 메트릭 레지스트리, 빌드 정보(`-ldflags`)

### 1.5 패키지 구조
- **`modules/<name>/`**: 독립 패키지. 외부 노출은 `modules/<name>/api.go` 또는 `pkg.go`만
- **`modules/<name>/internal/`**: 모듈 내부. 다른 모듈에서 import 금지
- **`pkg/`**: 모든 모듈이 쓰는 공통만. 신규 추가 시 ADR 필수

### 1.6 주석/문서
- 공개 API(`Capitalized` 식별자)에 GoDoc 주석 필수
- 무엇(WHAT)이 아니라 왜(WHY)/제약(CONSTRAINT) 위주
- 한 줄 설명으로 충분하면 한 줄

---

## 2. TypeScript / Vue 3

### 2.1 일반
- **TypeScript**: `strict: true` 강제
- **빌드**: Vite
- **컴포넌트**: `<script setup lang="ts">` 사용
- **상태**: `ref` / `reactive`. 컴포지션 함수(`useXxx`)로 캡슐화

### 2.2 API 호출
- 직접 `fetch` 금지. `composables/useApi.ts` 경유
- 모든 에러 토스트/리커버리는 공통 핸들러를 통해

### 2.3 i18n
- 모든 사용자 노출 문자열은 `t('...')` 사용 (ko/en)
- 한국어를 1급 시민으로 — 키 작명도 한국어 의미 반영

### 2.4 린트
- ESLint + Prettier 표준 설정
- `unused-imports`, `no-explicit-any` 활성화

---

## 3. 명명 규약 (전 언어 공통)

| 대상 | 규칙 | 예시 |
|---|---|---|
| Go 패키지명 | 소문자, 짧고 의미있게 | `storage` (NOT `data_storage`) |
| Go 파일명 | `snake_case.go` | `service_map.go` |
| Go 식별자 | `MixedCaps` / `mixedCaps` | `ServiceMap`, `parseInt` |
| Vue 컴포넌트 | `PascalCase.vue` | `ServiceMap.vue` |
| TS 변수/함수 | `camelCase` | `fetchMetrics` |
| 모듈 이름 (디렉터리) | `kebab-case` | `agent-node`, `agent-db-postgres` |
| 환경변수 | `COROOT_*` prefix | `COROOT_LISTEN_ADDR` |
| 메트릭 이름 | `coroot_*` prefix | `coroot_active_connections` |
| K8s 리소스명 | `kebab-case` | `coroot-cluster-agent` |

---

## 4. 컨피그 / 환경변수

- 모든 시크릿: 환경변수 또는 사내 Vault. 코드 하드코딩 금지
- `.env` 파일: dev 전용. `.env.example`만 커밋
- 환경변수 prefix: 반드시 `COROOT_`

---

## 5. 외부 호출 (HTTP/gRPC/DB)

- timeout 명시 (기본값 의존 금지)
- retry + circuit breaker는 `pkg/httpx`, `pkg/dbx` 사용
- proxy 환경변수(`HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY`) 반드시 존중
