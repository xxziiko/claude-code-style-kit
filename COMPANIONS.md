# Companion Plugin Strategy

`xxziiko`는 모든 일을 대신하는 플러그인이 아니라, 작업을 진행하는 기본 흐름과 품질 기준을 잡는 코어 키트입니다.

외부 플러그인은 역할이 겹치지 않게 아래처럼 보조 렌즈로 사용합니다.

## Core Flow

기본 작업은 항상 `xxziiko` 흐름으로 시작하고 끝냅니다.

```text
/xxziiko:scan
/xxziiko:do
/xxziiko:fix
/xxziiko:check
```

역할:

- `scan`: 프로젝트 구조, 요구사항, 성공 기준, 기존 패턴 파악
- `do`: 기존 패턴 우선, 새 의존성 없이 최소 변경 구현
- `fix`: 로그와 재현 증거 기반 디버깅
- `check`: 변경 요약, 검증 결과, 남은 리스크 정리

## Recommended Companions

### Ponytail

역할: 과설계 방지, 삭제 우선, 표준 라이브러리/기존 코드 재사용, 최소 diff 유지.

사용 타이밍:

- 구현 전: 요구사항이 실제로 필요한지, 기존 코드로 충분한지 확인
- 구현 중: 새 abstraction, 새 dependency, boilerplate가 생기려 할 때 제동
- 구현 후: 현재 diff에서 줄일 수 있는 코드 확인

권장 사용:

```text
/ponytail
/ponytail-review
/ponytail-audit
/ponytail-debt
```

`xxziiko`와의 조합:

```text
/xxziiko:scan
/ponytail
/xxziiko:do
/ponytail-review
/xxziiko:check
```

주의:

- Ponytail은 over-engineering을 줄이는 렌즈입니다.
- correctness, accessibility, security, data loss risk는 줄이면 안 됩니다.
- 의도적으로 단순화한 부분에 한계가 있으면 `ponytail:` comment로 ceiling과 upgrade trigger를 남깁니다.

### Frontend Fundamentals

역할: 프론트엔드 품질 렌즈. 사용자 경험, 접근성, 상태 흐름, 컴포넌트 책임, 프론트엔드 기본기를 점검합니다.

사용 타이밍:

- UI 동작 변경 후
- form, loading/error/empty state, interaction이 있는 작업 후
- 접근성이나 사용성 리스크가 있는 화면 변경 후

`xxziiko`와의 조합:

```text
/xxziiko:scan
/xxziiko:do
frontend-fundamentals review
/xxziiko:check
```

### Oh My Claude Code

역할: 여러 에이전트를 병렬로 운용하거나 긴 작업을 오케스트레이션합니다.

사용 타이밍:

- 관련 파일 탐색, 리뷰, 테스트 전략을 병렬로 나누면 속도가 날 때
- 작업 범위가 넓고 독립된 lane이 있을 때
- 단일 파일 수정이나 작은 버그 수정에는 사용하지 않습니다.

권장 분담:

- main session: 구현 판단과 최종 통합
- explorer: 관련 파일과 기존 패턴 탐색
- reviewer/verifier: diff와 검증 증거 확인

## Practical Playbooks

### Small Feature

```text
/xxziiko:scan
/ponytail
/xxziiko:do
/xxziiko:check
```

목표: 새 구조를 만들기 전에 기존 패턴으로 충분한지 확인하고, 가장 작은 기능 diff를 남깁니다.

### Bug Fix

```text
/xxziiko:scan
/xxziiko:fix
/xxziiko:check
```

목표: 증상 패치가 아니라 root cause를 고치고, 실패했던 검증을 다시 실행합니다.

### UI / Form / Interaction

```text
/xxziiko:scan
/xxziiko:do
frontend-fundamentals review
/xxziiko:check
```

목표: 구현 후 loading, error, empty, disabled, keyboard, accessibility 상태를 놓치지 않습니다.

### Refactor / Cleanup

```text
/xxziiko:scan
/ponytail-review
/xxziiko:do
/xxziiko:check
```

목표: 추가보다 삭제를 우선하고, behavior를 바꾸지 않는 작은 정리를 합니다.

### Larger Work

```text
/xxziiko:scan
oh-my-claudecode explorer/reviewer lanes
/xxziiko:do
/xxziiko:check
```

목표: 병렬 탐색과 리뷰는 외부 오케스트레이션에 맡기되, 구현 판단과 완료 판정은 `xxziiko` 흐름으로 고정합니다.

## Decision Rule

플러그인을 많이 켜는 것이 목표가 아닙니다.

- 작업 흐름과 완료 기준: `xxziiko`
- 과설계 방지: `ponytail`
- 프론트엔드 UX/기본기 점검: `frontend-fundamentals`
- 병렬 에이전트 운용: `oh-my-claudecode`

겹치면 `xxziiko`가 우선입니다. 외부 플러그인은 특정 관점의 리뷰와 제동 장치로만 사용합니다.

