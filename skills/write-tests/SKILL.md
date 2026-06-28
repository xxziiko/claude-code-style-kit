---
name: write-tests
description: 범용 테스트 작성 스킬. `ready-for-write-tests`로 판정된 테스트 매트릭스를 바탕으로 mode(tdd|regression|hardening)와 intent(behavior|contract|connectivity)에 맞는 테스트 코드를 작성하고 실행할 때 사용.
allowed-tools: Read, Grep, Glob, Edit, Write
---

# Write Tests

이 스킬은 **`ready-for-write-tests`로 판정된 테스트 설계**를 실제 테스트 코드로 바꿉니다.

핵심 목적:

1. 설계 없는 테스트 생성을 막는다.
2. 테스트를 기능 명세와 사용자 가치에 맞게 안정적으로 작성한다.
3. RED/GREEN 또는 회귀 보호막 여부를 실제 실행으로 확인한다.

## 1. 선행 조건

기본적으로 아래 중 하나가 있어야 합니다.

- `design-tests`가 `ready-for-write-tests`로 판정한 테스트 매트릭스
- 기존 테스트 보강 작업에서 충분히 구체적인 테스트 지시
- 이미 실패하거나 flaky한 테스트의 원인이 충분히 좁혀진 상태

설계가 없거나 모호하면 먼저 `design-tests`를 실행하고 멈춥니다.
regression에서는 matrix가 사용자 관찰 가능 behavior evidence를 포함해야 합니다.

## 2. Modes

### `tdd`

- 실패하는 테스트를 먼저 쓴다.
- RED를 실제로 확인한다.
- 이후 구현 단계로 handoff한다.

### `regression`

- 현재 동작을 고정하는 테스트를 쓴다.
- 변경 전에도 통과 가능하거나, 최소한 기대하는 보호막이 무엇인지 분명히 한다.

### `hardening`

- flaky 원인을 줄이는 방향으로 테스트를 고친다.
- sleep/retry 같은 masking fix를 지양한다.

## 3. Intents

### `behavior`

- 사용자 행동, 화면 상태, orchestration 회귀를 검증한다.
- MSW/mock 기반 integration test를 적극 사용할 수 있다.

### `contract`

- DTO -> Domain 변환, adapter, mapper 정합성을 검증한다.
- typed fixture, mapper assertions, typecheck가 우선이다.
- MSW는 보조 수단일 뿐이다.
- 단순 generated SDK wrapper 호출, React Query option shape, MSW handler 단독 호출, 필드 복사만 있는
  adapter smoke test는 작성하지 않는다. contract intent는 복잡한 mapper, normalization, 계산,
  error classification처럼 독립 로직을 검증할 실익이 있을 때만 사용한다.

### `connectivity`

- 실제 서버와의 연결 검증이 필요할 때 사용한다.
- 이 경우 테스트 파일보다 smoke/live verification 증거가 더 중요할 수 있다.

## 4. 이 스킬이 하는 일

1. `ready-for-write-tests` 테스트 설계를 읽는다.
2. 기존 테스트 패턴과 유틸을 탐색한다.
3. 테스트 코드를 작성한다.
4. 실행해 RED/GREEN 또는 회귀 통과 여부를 확인한다.
5. 실패 원인을 구조화해 보고한다.

## 5. 이 스킬이 하지 않는 일

- 설계 없는 테스트 범위 확장
- 테스트를 위해 제품 코드를 과도하게 왜곡
- 구현 코드를 슬쩍 고쳐 테스트를 맞추는 것
- 의미 없는 wiring 테스트 대량 생성
- 라이브러리 사용법 테스트 또는 adapter smoke test 생성

## 6. 작성 원칙

테스트 작성 전에 `${CLAUDE_PLUGIN_ROOT}/standards/test-driven-generation-workflow.md`의
“테스트 이름과 description 기준” 및 참고 자료를 확인한다. 테스트 이름은
Testing Library Guiding Principles, Kent C. Dodds의 implementation detail
테스트 글, Playwright/Cypress best practices의 방향처럼 레이어별 intent에 맞는
결과 언어를 사용한다. behavior/integration/E2E 테스트는 사용자 관찰 가능 결과를,
unit/contract 테스트는 계층 책임과 계약 결과를 드러내야 한다.

테스트 코드를 작성하기 직전에 `ready-for-write-tests` 테스트 매트릭스의 각 테스트명을 한 번 더
검토한다. 테스트명이 내부 타입, 필드, DTO, hook 반환 객체, scope 구성, column
shape, callback 호출 여부만 설명한다면 그대로 작성하지 말고 사용자/시스템 행동
문장으로 바꾼 뒤 작성한다. 테스트 본문에서는 필요한 내부 값을 assertion할 수
있지만, 테스트 description은 보호하려는 동작을 말해야 한다.

### MUST 1. 한 테스트는 한 behavior

한 테스트에서 여러 정책을 동시에 검증하지 않는다.

### MUST 2. 테스트 이름은 정책을 드러냄

테스트 이름은 구현이 아니라 기대 behavior를 드러낸다.

권장:

- `검색어를 입력해도 검색 실행 전까지 결과 조건이 바뀌지 않는다`
- `검색을 실행하면 입력한 검색어로 결과 조건이 갱신된다`
- `다른 상세를 열면 이전 상세 선택이 남아 있지 않다`
- `상세 수량을 선택하면 해당 기준의 개별 목록을 조회한다`
- `선택 항목이 없으면 선택 항목 전용 액션을 사용할 수 없다`
- `내보내기 실행 시 선택된 다운로드 유형을 전달한다`

피한다:

- `QueryCondition.keyword가 변경된다`
- `QueryCondition에 groupBy와 groupValue가 포함된다`
- `ExportScope가 현재 queryCondition을 mirror한다`
- `row가 previousQuantity와 totalQuantity를 가진다`
- `onSubmit이 호출된다`

내부 타입명이나 필드명은 테스트 본문에서 최소한으로 확인한다. description의
주어는 사용자가 기대하는 결과, 업무 정책, 또는 시스템 관찰 가능 동작이어야 한다.

### MUST 3. 기존 패턴 우선

기존 테스트 유틸, 렌더 함수, MSW/mock 패턴을 우선 재사용한다.

### MUST 4. intent에 맞는 증거를 만들어야 함

- `behavior` -> 사용자 관찰 가능 결과
- `contract` -> DTO/adapter 정합성
- `connectivity` -> live smoke 또는 동등한 연결 증거

contract 테스트를 작성할 때는 request body와 response envelope/data path를 함께 고정한다.
예를 들어 실제 응답이 `{ success, code, message, data: { ... } }`이면 mapper test도
같은 envelope을 사용하고, `data` 내부 값을 domain result로 매핑하는지 검증한다.
`wrapper 제거`라는 표현만 보고 외부 스키마 wrapper와 공통 envelope을 함께 제거하지 않는다.

### MUST 5. mode에 맞는 실행 결과 확인

- `tdd` -> 먼저 실패를 확인
- `regression` -> 보호막으로 통과 여부 확인
- `hardening` -> flaky 재현/개선 여부 확인

`regression` mode에서 테스트 실행 결과가 실패하거나 실행되지 않았으면 구현 완료를 주장하지 않는다.
contract 테스트만 작성했다면 실행이 통과해도 behavior regression을 보호했다고 주장하지 않는다.

### MUST 6. anti-pattern 금지

아래는 기본적으로 금지한다.

- queryOptions/mutationOptions 존재만 확인
- 단순 import/export 확인
- 라이브러리 내부 로직 재검증
- generated SDK wrapper가 호출되는지만 확인
- MSW handler를 직접 호출하거나 API wrapper가 mock data를 반환하는지만 확인
- DTO field를 그대로 복사하는 mapper/adapter만 확인
- 깨지기 쉬운 DOM 구조 결합 테스트
- `connectivity` intent인데 MSW 결과만으로 실제 연결 안정성을 주장
- 성공 콜백 호출만 확인하고 response result 값 또는 후속 refetch/invalidation을 검증하지 않는 테스트

## 7. 출력 형식

최소 출력 형식:

```txt
Mode:
- <tdd|regression|hardening>

Intent:
- <behavior|contract|connectivity>

Tests written:
- <file>: <behaviors>

Execution:
- command: ...
- result: <red|green|pass|fail>

Coverage notes:
- protected behaviors: ...
- remaining gaps: ...
- evidence type: <mocked behavior|typed contract|live connectivity>

Decision blockers:
- ...

Agent gate:
- status: <ready-for-implementation|blocked-by-test-failure|blocked-by-low-value-test|blocked-on-decision>
- reason: ...
```

## 8. 좋은 결과의 기준

좋은 결과는 아래를 만족한다.

1. `ready-for-write-tests` 설계 범위만 구현한다.
2. 테스트가 사용자 행동이나 정책을 검증한다.
3. 실행 결과가 신선한 출력으로 확인된다.
4. 남은 테스트 공백이 있으면 이유와 위험을 적는다.
5. 이후 구현 에이전트가 그대로 활용할 수 있다.
6. intent와 evidence type이 일치한다.

## 9. 연결 스킬

- 테스트 전략 설계는 `design-tests`
- 기능 구현은 별도 구현 워크플로우가 담당
