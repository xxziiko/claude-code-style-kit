---
name: design-tests
description: 범용 테스트 설계 스킬. 신규 기능, 버그 수정, 리팩토링, 서버 상태 연결 작업에서 mode(tdd|regression|hardening)와 intent(behavior|contract|connectivity) 기준으로 테스트 전략과 테스트 매트릭스를 설계하고 다음 단계 진입 가능 여부를 판정할 때 사용.
allowed-tools: Read, Grep, Glob, Edit, Write
---

# Design Tests

이 스킬은 **테스트를 작성하지 않고**, 먼저 무엇을 어떤 레이어에서 검증해야 하는지 설계합니다.

핵심 목적:

1. AI가 low-value 테스트를 만들기 전에 테스트 가치를 먼저 좁힌다.
2. 신규 기능과 회귀 작업을 같은 표면에서 다루되, mode로 분기한다.
3. 테스트 작성 전에 에이전트가 `ready-for-write-tests` 또는 `blocked-by-design`을 판정할 수 있는 산출물을 만든다.

## 1. 사용 시점

아래 상황에서 사용합니다.

- 신규 feature 작업의 TDD 시작 전
- 기존 기능 변경 전 회귀 테스트 설계
- flaky 테스트 정리 전 테스트 전략 재설계
- UI와 서버 상태 연결, adapter 교체, 데이터 경계 변경 전 테스트 전략 수립

## 2. Modes

### `tdd`

신규 기능 또는 명세 기반 작업에서 사용합니다.

목표:

- acceptance behavior를 먼저 고정
- RED 테스트 우선순위 정의
- 구현 이전에 테스트 경계를 좁힘

### `regression`

기존 기능 변경, 서버 상태 연결, 리팩토링에서 사용합니다.

목표:

- 현재 사용자 동작을 먼저 고정
- 변경으로 깨지면 안 되는 behavior 식별
- 구현 세부가 아니라 사용자 관찰 가능 결과에 집중

### `hardening`

불안정하거나 flaky한 테스트를 다룰 때 사용합니다.

목표:

- flaky root cause 가설 정리
- 안정적인 레이어로 테스트 재배치
- 불필요한 중복 테스트 제거

## 3. Intents

### `behavior`

- 사용자 행동, 화면 상태, orchestration 회귀를 보호
- component/hook integration, browser/E2E, MSW/mock 기반 검증이 주력
- 사용자 관찰 가능 결과가 gate evidence다.

### `contract`

- DTO -> Domain 변환, adapter, mapper, type contract 정합성 보호
- typed fixture, mapper assertions, typecheck가 주력
- MSW는 보조 수단
- contract evidence는 behavior gate를 대체하지 않는다.

### `connectivity`

- 실제 서버와의 연결 안정성 확인
- live smoke / staging verification이 주력
- MSW는 최종 증거가 아님

## 4. 이 스킬이 하는 일

1. 명세, 변경 범위, 기존 테스트를 읽는다.
2. 보호해야 할 사용자 가치와 비즈니스 정책을 식별한다.
3. 어떤 레이어에서 테스트할지 결정한다.
   - unit
   - integration
   - e2e
   - live smoke
4. 테스트 매트릭스를 만든다.
5. low-value / anti-pattern 테스트를 명시적으로 제외한다.
6. 다음 단계 진입 가능 여부를 판정하고, 필요한 경우 의사결정 blocker를 분리한다.

## 5. 이 스킬이 하지 않는 일

- 테스트 코드 작성
- 기능 구현
- readiness 판정 없이 테스트 범위 확대
- 라이브러리 동작 재검증을 위한 테스트 설계
- `write-tests` 자동 실행

## 6. 설계 원칙

테스트 설계 전에 `${CLAUDE_PLUGIN_ROOT}/standards/test-driven-generation-workflow.md`의
“테스트 이름과 description 기준” 및 참고 자료를 확인한다. 특히
Testing Library Guiding Principles, Kent C. Dodds의 implementation detail
테스트 글, Playwright/Cypress best practices를 기준으로 behavior/integration/E2E
테스트가 사용자 관찰 가능 결과를 보호하는지 점검한다.

### MUST 1. 사용자 가치 기준

테스트는 구현 디테일이 아니라 아래 중 하나를 보호해야 한다.

- 사용자 행동
- 비즈니스 정책
- 상태 전이
- 화면 후처리

### MUST 2. 명세 범위 준수

명세 또는 변경 범위를 벗어난 테스트를 제안하지 않는다.

### MUST 3. low-value 테스트 배제

아래는 기본적으로 배제한다.

- 라이브러리 자체 동작 검증
- queryOptions/mutationOptions 존재만 확인하는 테스트
- SDK wrapper 성공/실패 message 전달만 확인하는 테스트
- 구현 세부에 과도하게 결합된 snapshot-like 테스트
- 단순 DTO field copy만 확인하는 mapper 테스트

### MUST 4. 가장 가까운 레이어 우선

기본 우선순위:

1. component/hook integration
2. domain orchestration 또는 adapter contract
3. e2e

진짜 domain 규칙이 독립적으로 존재할 때만 unit test를 우선한다.

### MUST 5. intent를 먼저 분리

테스트를 설계할 때는 반드시 아래 중 무엇을 증명하려는지 먼저 분리한다.

- behavior
- contract
- connectivity

이 intent가 없으면 mock 사용 의미나 테스트 가치가 흐려진다.

### MUST 6. 에이전트가 판정 가능한 산출물

출력은 에이전트가 후속 `write-tests` 진입 여부를 판정할 수 있는 테스트 매트릭스여야 한다.

### MUST 7. write-tests readiness gate

`design-tests`는 테스트 코드를 작성하지 않는다.
대신 마지막에 아래 중 하나로 판정한다.

- `ready-for-write-tests`: 테스트 매트릭스가 구체적이고, low-value 테스트가 배제됐고, 필요한 검증 기준이 명확하다.
- `blocked-by-design`: 보호할 behavior, 레이어, fixture source, 기존 테스트 패턴 중 하나가 부족하다.
- `blocked-on-decision`: 에이전트가 임의로 정하면 설계 방향이 바뀌는 의사결정이 남아 있다.

`blocked-by-design` 또는 `blocked-on-decision` 상태에서는 `write-tests`로 넘어가지 않는다.

## 7. 출력 형식

최소 출력 형식:

```txt
Mode:
- <tdd|regression|hardening>

Intent:
- <behavior|contract|connectivity>

Scope:
- spec: ...
- changed surfaces: ...

Protected behaviors:
- ...

Recommended layers:
- integration: ...
- e2e: ...
- unit: ...
- live smoke: ...

Execution strategy:
- query: <suspense-first|non-suspense-exception|not-applicable>
- mutation: <mutate-first|mutateAsync-exception|not-applicable>
- reason: ...

Test matrix:
- [P0] <surface> -> <protected behavior> -> <layer> -> evidence: <behavior|contract|connectivity> -> <why>
- [P1] <surface> -> <protected behavior> -> <layer> -> evidence: <behavior|contract|connectivity> -> <why>

Gate evidence:
- user-facing behavior: <yes|no|n/a>
- contract-only tests excluded from behavior gate: <yes|no|n/a>
- fixture source: <existing test fixture|typed fixture|sample response|backend contract|n/a>
- validator: <command or n/a>

Excluded tests:
- <test idea> -> <why low-value>

Agent gate:
- status: <ready-for-write-tests|blocked-by-design|blocked-on-decision>
- reason: ...
- next skill: <write-tests|none>

Decision blockers:
- ...
```
