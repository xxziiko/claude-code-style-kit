# Test-Driven Generation Workflow

> AI 기반 기능 개발과 API 전환 작업에서 테스트를 먼저 설계하고, 그 설계를 기준으로 안정적으로 테스트와 구현을 진행하기 위한 표준 워크플로우

- Status: Draft
- Date: 2026-04-25
- Scope: Repo-wide

## 1. 목적

이 문서는 AI가 생성한 코드의 신뢰성을 높이기 위해 테스트를 먼저 설계하고,
그 설계를 기준으로 구현을 통제하는 방법을 정의한다.

핵심 목표:

1. low-value 테스트를 줄인다.
2. 명세와 테스트를 먼저 정렬한다.
3. 구현 전에 회귀 보호막 또는 RED 테스트를 확보한다.
4. AI 생성 코드의 품질을 테스트로 통제한다.

## 2. 기본 원칙

1. 명세가 먼저다.
2. 테스트 설계와 테스트 작성은 분리한다.
3. 설계 없는 테스트 작성은 예외적으로만 허용한다.
4. 사용자 행동과 정책을 보호하는 테스트를 우선한다.
5. 기존 기능 변경은 regression-first, 신규 기능은 test-plan-first를 기본으로 한다.
6. 테스트는 intent(behavior|contract|connectivity)를 먼저 구분한다.

## 3. 범용 테스트 워크플로우

### `design-tests`

역할:

- 명세/변경 범위를 읽고 테스트 전략을 설계
- 무엇을 어느 레이어에서 검증할지 결정
- low-value 테스트를 제외
- 에이전트가 다음 단계 진입 가능 여부를 판정할 수 있는 테스트 매트릭스 생성

지원 mode:

- `tdd`
- `regression`
- `hardening`

지원 intent:

- `behavior`
- `contract`
- `connectivity`

### `write-tests`

역할:

- `ready-for-write-tests`로 판정된 테스트 설계를 실제 테스트 코드로 작성
- RED/GREEN 또는 회귀 보호막 여부를 실행으로 확인
- flaky risk와 남은 공백을 보고

지원 mode:

- `tdd`
- `regression`
- `hardening`

지원 intent:

- `behavior`
- `contract`
- `connectivity`

## 4. 권장 순서

### 신규 feature

1. 명세 작성/정리
2. `design-tests --mode tdd --intent behavior`
3. `Agent gate.status: ready-for-write-tests` 판정
4. `write-tests --mode tdd --intent behavior`
5. 구현
6. 리팩토링

### 기존 기능 변경 / API 전환

1. 변경 범위 파악
2. `design-tests --mode regression --intent behavior`
3. `Agent gate.status: ready-for-write-tests` 판정
4. `write-tests --mode regression --intent behavior`
5. 실제 호출부/기능 변경
6. 회귀 검증

### flaky 테스트 정리

1. 문제 테스트 수집
2. `design-tests --mode hardening --intent behavior`
3. `write-tests --mode hardening --intent behavior`
4. 안정화 확인

## 5. scaffold-api와의 연결

REST API 도입 워크플로우에서는 아래 순서를 권장한다.

1. `scaffold-api`
   - domain proposal gate가 `blocked`이면 중단
2. `design-tests --mode regression --intent behavior`
   - `ready-for-write-tests` 판정 전에는 중단
3. `write-tests --mode regression --intent behavior`
   - regression 통과 전에는 caller migration 금지
4. `connect-api`
5. 최종 회귀 검증

실제 오케스트레이션 보조 명령:

- `yarn create-api:workflow`
- `yarn scaffold-api:workflow --service <wms|oms>`

의미:

- `create-api:workflow`는 generated SDK에는 있지만 `src/`에서 import되지 않은 operation을 candidate로 보여주고 선택 지점을 만든다.
- `scaffold-api`는 domain 자산 생성과 deterministic implementation에 집중한다.
- 사용자 화면 보호막은 `design-tests` / `write-tests`가 만든다.
- caller migration은 보호막이 준비된 뒤에만 진행한다.
- `Next agent actions`는 다음 후보 작업이지 gate 판정을 우회하는 자동 실행 허가가 아니다.
- caller migration 보호막은 caller-facing behavior여야 한다.
- domain API 직접 호출, DTO mapper, queryOptions/mutationOptions, mutationFn 성공 테스트는 보조 contract evidence일 수 있지만 caller migration gate를 열 수 없다.
- API caller migration regression 설계는 `yarn validate:api-caller-migration-design <artifact.json>`가 `passed`여야 `ready-for-write-tests` matrix로 취급한다.
- 실제 API 연결 안정성은 behavior regression과 별도 intent로 다룬다.

### `api-smoke`

역할:

- 실제 서버와의 연결 안정성을 검증
- behavior regression과 별도의 connectivity evidence 제공

핵심 규칙:

- `connectivity` intent의 최종 증거는 MSW가 아니라 live smoke다
- mock 기반 통과만으로 실제 API 연결 안정성을 주장하지 않는다

### `connect-api`

역할:

- 기존 GraphQL/legacy/직접 호출부를 새 domain query/mutation으로 교체
- query invalidation과 alert/navigation 같은 사용자 후처리 보존
- regression 통과 후에만 `integrated` 상태 판정

핵심 규칙:

- regression tests ready 이전에는 caller migration 금지
- handoff는 `blocked-on-decision`처럼 에이전트가 더 진행할 수 없는 의사결정 blocker만 남았을 때만 가능

## 6. 좋은 테스트 설계 기준

좋은 테스트 설계는 아래를 만족한다.

1. 보호할 behavior가 명확하다.
2. 왜 unit/integration/e2e 중 그 레이어인지 설명할 수 있다.
3. 라이브러리 동작 재검증 같은 low-value 테스트를 배제한다.
4. 다음 단계 진입 가능 여부가 `ready-for-write-tests`, `blocked-by-design`, `blocked-on-decision` 중 하나로 정리되어 있다.

## 7. 좋은 테스트 작성 기준

좋은 테스트 작성은 아래를 만족한다.

1. 각 테스트가 한 behavior만 검증한다.
2. 테스트 이름이 기대 결과를 설명한다.
3. 실행 결과가 신선한 출력으로 확인된다.
4. 기존 테스트 유틸과 패턴을 재사용한다.
5. 구현 세부보다 사용자 관찰 가능 결과를 검증한다.
6. 테스트 description은 내부 타입/필드/DTO shape가 아니라 보호하려는 행동을 말한다.

### 테스트 이름과 description 기준

테스트 이름은 테스트 레이어와 intent에 맞는 결과 언어를 사용한다.

#### Behavior / integration / E2E 테스트

사용자 행동, 화면 상태, orchestration 회귀를 보호하는 테스트는 사용자 또는
시스템 관찰 가능 결과로 이름을 쓴다.

권장:

- `주문수정 저장 후 변경된 수령인이 목록에 반영된다`
- `주문확인 전환 성공 후 선택한 주문이 목록에서 제거되고 선택이 해제된다`
- `필터 조건이 바뀌면 기존 선택 상태가 해제된다`
- `검색어를 입력해도 검색 실행 전까지 결과 조건이 바뀌지 않는다`
- `상세 수량을 선택하면 해당 기준의 개별 재고 목록을 조회한다`
- `선택 품목이 없으면 특정 품목 상세 다운로드를 사용할 수 없다`

피한다:

- `refetchQueries 가 호출된다`
- `mutation 페이로드에 전달된다`
- `onSave 후 refetch 호출`
- `StockQueryCondition.keyword 가 변경된다`
- `StockExportScope 가 현재 queryCondition 을 mirror 한다`
- `DataTable row 가 totalQuantity 필드를 가진다`

이런 구현 중심 표현은 테스트 본문 assertion으로는 필요할 수 있지만, behavior 테스트
description의 주어가 되면 안 된다. 해당 테스트가 실제로 보호하는 사용자 결과가
없다면 테스트 레이어가 잘못되었거나 low-value 테스트일 가능성이 높다.

테스트 설계가 내부 타입명 중심으로 작성되어 있더라도 `write-tests` 단계에서는
그대로 옮기지 않는다. 먼저 description을 행동 문장으로 바꾼다. 예를 들어
`StockUnitQueryCondition에 groupBy와 groupValue가 포함된다`는
`상세 수량을 선택하면 해당 기준의 개별 재고 목록을 조회한다`로 바꾼 뒤,
본문 assertion에서 필요한 query condition 값을 확인한다.

#### Contract / unit 테스트

adapter, mapper, domain API, hook의 좁은 책임을 검증하는 테스트는 내부 함수명을
그대로 반복하기보다 계층 책임을 설명한다. 단위 테스트라서 내부 호출을 검증해야
하는 경우에도 description은 호출 세부보다 계약 결과를 우선한다.

권장:

- `주문 상태 변경 성공 후 주문 목록 갱신을 요청한다`
- `주문취소 성공 후 주문 목록 갱신을 요청한다`
- `상태변경 command 응답의 성공 여부와 메시지를 반환한다`

피한다:

- `updateOrdersStatusMutation 성공 시 apolloClient.refetchQueries 가 GET_COLLECTED_ORDERS 로 호출된다`
- `cancelOrdersMutation 성공 시 apolloClient.refetchQueries 가 GET_COLLECTED_ORDERS 로 호출된다`

예외적으로 migration bridge처럼 임시 구현 책임 자체가 보호 대상이면 assertion에서
`refetchQueries({ include: [GET_COLLECTED_ORDERS] })`를 확인할 수 있다. 그래도
테스트 이름은 `목록 갱신을 요청한다`처럼 유지보수자가 읽는 업무 책임으로 표현한다.

### 참고 자료

이 기준은 아래 공개 테스트 가이드의 원칙을 프로젝트 문맥에 맞게 적용한다.
테스트 설계나 작성 중 테스트 이름, assertion 레이어, 구현 세부 결합 여부가 애매하면
아래 자료를 먼저 확인하고 이 문서의 레이어별 기준으로 구체화한다.

- [Testing Library Guiding Principles](https://testing-library.com/docs/guiding-principles): 테스트는 소프트웨어가 사용되는 방식과 닮을수록 신뢰도가 높다는 원칙.
- [Kent C. Dodds - Testing Implementation Details](https://kentcdodds.com/blog/testing-implementation-details): 구현 세부 테스트가 리팩토링에 취약하고 실제 회귀를 놓칠 수 있는 이유.
- [Playwright Best Practices](https://playwright.dev/docs/best-practices): E2E 테스트는 사용자에게 보이는 동작과 안정적인 locator를 기준으로 작성한다는 공식 가이드.
- [Cypress Best Practices](https://docs.cypress.io/app/core-concepts/best-practices): E2E 테스트의 사용자 흐름, selector 안정성, 독립성에 대한 참고 가이드.

## 8. 피해야 할 것

- 구현 후에 테스트를 끼워 맞추기
- queryOptions/mutationOptions 존재만 확인하는 테스트
- SDK wrapper 성공/실패 message 전달만 검증하는 테스트
- sleep/retry로 flaky를 덮는 테스트
- 설계 없이 AI에게 테스트 코드를 바로 생성시키는 것

## 9. 한 줄 정의

이 워크플로우는 "AI가 코드를 잘 쓰길 기대하는 방식"이 아니라
"명세와 테스트를 먼저 고정해 AI가 잘못 써도 잡아내는 방식"이다.
