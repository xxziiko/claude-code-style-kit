# Shared UI 컨벤션

Status: Active
Updated: 2026-05-18

## 목적

이 문서는 Sirloin OMS의 official shared UI 기준을 정의하는 source of truth다.

목표는 기존 컴포넌트를 한 번에 모두 이동하는 것이 아니다. 새로 만들거나 승격하는
공통 UI를 예측 가능하게 관리하는 것이 목표다. 읽는 사람이 위치, official 기준,
도메인 소유 여부, legacy 유지 여부를 다시 생각하지 않도록 한다.

브랜치별 실행 계획, 조사 인벤토리, 오래된 판단 기록, superseded draft는 로컬
하네스에 둔다. 팀이 계속 따라야 하는 현재 표준만 이 저장소의 `docs/standards`
아래에 둔다.

## 목표 구조

official shared UI/hook/constant와 domain 계층의 장기 SSOT는 `src/components/**` 또는 `src/hooks/**`가 아니다.

```text
src/
  shared/                       # 공용 (cross-cutting reusable)
    ui/
      {ComponentName}/
        {ComponentName}.tsx
        {ComponentName}.stories.tsx
        {ComponentName}.test.tsx
    hooks/                      # 도메인 무관 UI/state 메커니즘
    constants/                  # 도메인 무관 공용 상수
    utils/                      # (장기) 도메인 무관 utility
  domain/
    {aggregate}/{capability}/
      mappers/
      rules/
      types/
  features/
    {aggregate}/{capability}/
      ui/
      hooks/                    # feature 결합 hook
      types.ts
      utils/                    # feature-local helper
```

도메인 하위 화면에 결합된 신규 훅은 재사용 여부와 무관하게
`src/features/{aggregate}/{capability}/hooks`에 둔다. 도메인 무관 공용 hook은
`src/shared/hooks`에 둔다. 도메인 정책·분류 규칙은 domain layer에 두고,
feature-local 표시/상호작용 helper는 `src/features/{aggregate}/{capability}/utils`에 둔다.
도메인 무관 공용 util은 `src/shared/utils`에 둔다.

## 배치 규칙

- 신규 official shared UI는 `src/shared/ui/{ComponentName}`에만 둔다.
- feature에 종속된 UI는 `src/features/{aggregate}/{capability}/ui`에 둔다.
- `src/components/**`는 과도기 legacy surface로 본다.
- 신규 official shared component를 `src/components/**` 아래에 추가하지 않는다.
- 기존 파일을 일괄 이동하지 않는다. 변경 대상 컴포넌트를 리뷰할 때 public
  contract가 명확한 경우에만 승격하거나 이관한다.
- `shared/common` 디렉토리는 만들지 않는다. `shared` 자체가 공통 계층이다.

다음 중 하나라도 알면 domain-aware UI다.

- 도메인 명사
- 도메인 상태
- 업무 흐름
- API/DTO 해석
- route/path
- permission
- 제품별 상태 전이

## Official Shared UI 기준

컴포넌트가 official shared UI가 되려면 아래 조건을 모두 만족해야 한다.

1. `src/shared/ui/{ComponentName}/{ComponentName}.tsx`에 위치한다.
2. 같은 디렉토리에 Storybook story가 있다.
3. 도메인 코드를 import하지 않는다.
4. fake data와 props만으로 Storybook에서 주요 상태를 보여줄 수 있다.
5. public API에 `sx`, `SxProps<Theme>`, 넓은 MUI props pass-through를 열지 않는다.
6. named export를 사용하고 barrel export를 만들지 않는다.

하나라도 빠지면 아직 official이 아니다.

| 상태                  | 의미                                            | 위치                            |
| --------------------- | ----------------------------------------------- | ------------------------------- |
| `official-shared-ui`  | 프로젝트 표준 공통 UI                           | `src/shared/ui/{ComponentName}` |
| `candidate-shared-ui` | official 기준을 아직 만족하지 못한 공통 UI 후보 | 기존 위치 또는 임시 PoC 위치    |
| `feature-ui`          | feature 언어와 업무 흐름을 아는 UI              | `src/features/{aggregate}/{capability}/ui` |
| `legacy-components`   | 리뷰 전까지 유지하는 과도기 표면                | `src/components/**`             |

## Storybook 규칙

Storybook은 official shared UI의 필수 조건이다.

- `{ComponentName}.stories.tsx`를 컴포넌트와 같은 디렉토리에 둔다.
- story title은 `shared/ui/{ComponentName}` 형식을 사용한다.
- fake data만 사용한다.
- GraphQL document, generated DTO, REST SDK, route constant, Recoil atom,
  domain context를 import하지 않는다.
- 기본 상태와 제품에서 반복되는 핵심 변형만 포함한다.
- story를 더 유연하게 만들기 위해 컴포넌트 API를 넓히지 않는다.

## 테스트 정책

shared UI의 테스트는 **Storybook이 1차 검증 수단**이다. 별도의 RTL/Vitest
테스트는 다음의 명시적 사유가 있을 때만 작성한다.

### 작성하지 않는다

- prop 값이 화면에 그대로 표시되는지만 확인하는 단순 표시 테스트
- `<ComponentName label="x" /> → screen.getByText('x')` 같이
  Storybook으로 시각 시연되는 것과 정보가 중복되는 테스트
- 도메인 무관 시각 컴포넌트(StatusChip, MetricCard 같은 표시 위주)
- 단순 prop pass-through만 검증하는 테스트

### 작성하는 경우만

- **상호작용 로직**: onValueChange가 정확한 value로 호출되는지, Enter 키
  처리, disabled 상태에서 callback 차단 같은 분기 로직
- **접근성 contract**: ariaLabel/label로 role을 찾을 수 있는지,
  selected/disabled aria-\* 속성, listbox/option role 동작
- **포맷팅·계산**: formatValue 결과, clamp/range, locale 포맷, 텍스트 변환
- **상태 전이**: variant별 분기, status 기반 표시 변경, 조건부 컴포넌트
- **회귀 가능성이 높은 contract**: 외부 라이브러리(MUI/Radix) 동작이
  변경되면 깨질 수 있는 부분, 또는 과거에 깨진 적이 있는 부분

### 판단 기준

테스트가 "이 컴포넌트가 변경됐을 때 깨져야 하는 의미 있는 케이스를
보호하는가?"를 자문한다. 답이 "Storybook으로도 충분히 보인다"라면
작성하지 않는다.

## Public API 규칙

- controlled component를 기본으로 둔다.
- 구현체 중심 MUI prop보다 제품 관점 prop 이름을 우선한다.
- callback prop은 React public API 관례에 맞춰 `on*` 이름을 사용한다.
- `handle*`는 컴포넌트/훅 내부 구현체 handler 이름으로만 사용하고 public prop에
  노출하지 않는다.
- `onChange`보다 의미가 명확하면 `onValueChange`, `onTextChange`,
  `onSelectionChange`를 사용한다.
- 접근성 이름은 `label`을 우선한다. visual label이 없을 때만 `ariaLabel`을 받는다.
- `disabled`, `placeholder`, `helperText`, `errorText`처럼 제품에서 반복되는 범용
  상태는 허용한다.
- `size`나 `variant`는 반복 사용이 확인된 뒤 제한된 union으로 추가한다.
- 도메인 계산, URL 상태, React Hook Form wiring, 서버 상태는 shared UI 밖에 둔다.

## 검색 입력 상태 소유권

- `SearchField`는 `value`, `onValueChange`, `onEnter`를 받는 controlled UI 컴포넌트다. 내부에서 검색 상태나 조회 조건을 소유하지 않는다.
- 리스트/조회 화면에서 입력 draft와 실제 조회 keyword를 분리해야 한다면 **submit-driven과 debounce 기반 즉시 필터링 모두 `useKeywordSearch`를 사용한다**. 같은 책임을 다루는 평행 훅(`useSearchKeyword*`, `useKeywordFilter*`, `useDebouncedKeyword*` 등)을 따로 만들지 않는다.
- URL search params, form-controlled 검색 같이 별도 상태 소유자가 명확한 경우는 해당 owner가 상태를 가진다.
- `SearchSection`은 `SearchField + 명시적 검색 버튼` 시각/인터랙션 묶음이 필요할 때만 사용한다. 버튼이 없는 즉시/debounce 검색은 `SearchField`를 직접 조합한다.

## 그룹화 규칙

현재 단계에서 official shared UI는 `src/shared/ui/{ComponentName}/` **단일
그룹, flat** 구조를 유지한다. 추상화 레벨이든 기능 분류든 하위 그룹 폴더는
도입하지 않는다.

```text
src/shared/ui/{ComponentName}/{ComponentName}.tsx
```

### 30초 자명성 룰

그룹을 도입하려면 그룹 이름만으로 "이 컴포넌트가 어느 그룹에 들어가는가"가
30초 안에 결정 가능해야 한다. `inputs` vs `display`, `pattern` vs `template`,
`navigation` vs `display`처럼 경계 고민이 생기는 그룹명은 도입하지 않는다.
경계 고민이 생긴다는 것 자체가 좋은 분류축이 아직 없다는 신호다.

### 명명 컨벤션이 1차 그룹 도구

폴더로 그룹화하기 전에 컴포넌트 이름의 prefix/suffix가 카테고리를 충분히
드러내는지 본다.

- `Filter*` — 필터 필드 (FilterSelectField, FilterDateRangePicker)
- `View*` — 보기 전환 (ViewTabs)
- `Search*` — 검색 입력 (SearchField)
- `Status*` — 상태 표시 (StatusChip)
- `*Card`, `*CardGroup` — 카드 표시
- `*Fallback` — 상태 시각화 합성 (LoadingFallback, EmptyFallback, ErrorFallback)

명명이 카테고리 의도를 드러내면 폴더 그룹은 미룬다.

### 재검토 트리거

다음 시점에 그룹화를 재검토한다.

- official shared UI가 **12개 이상**으로 평탄 구조가 IDE 탐색에서 실제로
  부담이 된다는 보고가 나올 때
- 같은 prefix/suffix가 **6개 이상** 반복되면서 그 prefix 그룹이 자명해질 때
- 새 컴포넌트 추가 시 위치 질문이 반복될 때

재검토 시점에 자명한 분류축이 나오면 도입하고, 안 나오면 계속 평탄 유지한다.

### Rationale — 왜 단일 그룹인가

이전 PoC 단계(`docs/common-components-poc-candidates.md`)에서는 컴포넌트를
`base UI / modal shell`, `layout pattern`, `table/page pattern`,
`filter/state pattern`, `state/fallback UI` 같은 추상화·책임 카테고리로
분류했고, 그 결과 `src/components/shared/patterns/{actionBar, dialog, fallback}/`
같은 디렉토리가 만들어졌다. 이번 단계에서 그 분류를 폐기하고 단일 그룹
`src/shared/ui/`로 통일한 이유는 다음과 같다.

**현재 단계의 트레이드오프**

| 항목                          | 카테고리 분리                               | 단일 그룹 (flat)                    |
| ----------------------------- | ------------------------------------------- | ----------------------------------- |
| 위치 결정 비용                | 추상화 레벨 판단 필요                       | **0 (항상 `src/shared/ui/{Name}`)** |
| 신규 컨트리뷰터 진입          | 카테고리 경계 학습 필요                     | **단일 패턴**                       |
| 카테고리 경계 모호성          | `StatusChip`이 ui? feedback? 같은 갈등 발생 | **30초 룰로 차단**                  |
| 컴포넌트 책임 변경 시 이동    | 카테고리 폴더 이동 비용                     | **이동 없음**                       |
| 컨벤션 강제 (lint)            | 다중 패턴 검증                              | **단일 패턴 검증**                  |
| 코드 리뷰어 자동 라우팅       | 카테고리별 가능                             | 컴포넌트별 지정                     |
| 컴포넌트 수가 50+로 누적될 때 | 의미 단위로 탐색 용이                       | IDE 탐색 부담                       |
| 의존성 가시화 (대규모 팀)     | cross-category import 명확                  | shared/ui 내부 자유                 |

소규모 팀 + 컴포넌트 수가 두 자릿수 초반 단계에서는 **단일 그룹의 진입 마찰
감소와 컨벤션 진화 비용 절감이 이득**이다. 대규모 팀과 50+ 컴포넌트 단계로
가면 카테고리 분리가 코드 리뷰 라우팅과 의존성 가시화에 유리해진다 — 그
시점에 재검토 트리거로 다시 판단한다.

**polylithic 분류는 왜 모호한가**

추상화 레벨 분류(`atoms/molecules/organisms`, `base/patterns/templates`)는
컴포넌트가 단순할 때는 자명하지만, 합성·상태·맥락이 섞이는 실제 컴포넌트는
경계가 흐릿하다. `LoadingFallback`은 단순 시각 요소(atom)인가 상태 표현
패턴(pattern)인가, `MetricCardGroup`은 합성(molecule)인가 그냥 ui인가 같은
질문이 매번 발생한다. 명명 컨벤션(`*Fallback`, `*CardGroup`)이 이 의도를
이미 드러내므로 폴더 분리는 중복 표현이다.

**현재 적용 결과**

- 6개 컴포넌트가 `src/shared/ui/{Name}/`에 평탄
- 명명 prefix/suffix가 카테고리 의도 1차 표현 (`Filter*`, `View*`, `*Card`)
- 컨벤션 위반 자동 감지 단순화 (단일 패턴만 검사)

## Template 위치

`ListPageTemplate` 같은 page/layout template은 지금 단계에서 official shared UI로
승격하지 않는다.

- template 표준이 2개 이상 반복되기 전까지 `src/shared/templates`를 만들지 않는다.
- page/layout 표준이 별도로 확정되기 전까지 기존 template은 legacy 위치에 둔다.
- template이 도메인 흐름, query orchestration, URL 상태, permission을 알면
  feature UI 또는 page layer에 둔다.
- template이 단순한 시각 구조만 담당하고 Storybook으로 fake data 시연이 가능해도,
  별도 template 표준 문서가 생기기 전에는 `src/shared/ui`로 올리지 않는다.

이 결정의 목적은 새 폴더를 미리 만들지 않는 것이다. 공식 shared UI의 첫 기준은
작고 반복되는 컴포넌트다.

## Shared Hooks / Utils / Constants 위치

hook, util, 상수는 UI 컴포넌트 디렉토리 아래에 두지 않는다. 도메인 무관 공용은
`src/shared/{hooks,utils,constants}/`에 평탄한 하위 폴더로 둔다.

- shared hook → `src/shared/hooks/`
- shared util → `src/shared/utils/` (장기, 기존 `src/utils/`는 legacy surface). feature-local util은 대신 `src/features/{aggregate}/{capability}/utils/`에 둔다.
- shared constant → `src/shared/constants/`
- 특정 feature에 결합된 hook은 shared hook이 아니므로 `src/features/{aggregate}/{capability}/hooks`에 둔다.
- 특정 도메인의 비즈니스 정책·규칙은 shared util이 아니므로 domain layer에 둔다.
- query/mutation hook, URL state hook, form adapter hook은 `src/shared/ui/**`에 두지 않는다.
- 기존 `src/hooks/**`, `src/utils/**`는 legacy surface다. 신규 추가는 역할에 따라 `src/shared/...`,
  `src/features/{aggregate}/{capability}/...`, 또는 `src/domain/{aggregate}/{capability}/...` 중 하나로 둔다.
  수정 중인 파일은 위 target 중 한 곳으로 점진 이관.

이 결정은 이전 "`src/shared/{hooks,utils}`는 만들지 않는다" 포지션을 대체한다 (`src/shared/constants/pagination.ts` 도입 이후 구조 일관성 확보).

## Feature / Domain 하위 구조

feature UI는 `src/features/{aggregate}/{capability}/ui`에 둔다. Domain layer는
API boundary, mapper, query/mutation contract, business rules를 소유한다.

- UI: `src/features/{aggregate}/{capability}/ui`
- feature hook: `src/features/{aggregate}/{capability}/hooks`
- feature type: `src/features/{aggregate}/{capability}/types.ts`
- domain 규칙: `src/domain/{aggregate}/{capability}/rules.ts`
- domain 타입: `src/domain/{aggregate}/{capability}/types/*`
- domain mapper: `src/domain/{aggregate}/{capability}/mappers.ts`
- smart constructor/business rule file: 구체적인 비즈니스 이름 사용
- domain query/mutation contract: `src/domain/{aggregate}/{capability}/{queries,mutations}.ts`

`src/hooks/{domain}`의 기존 훅은 legacy surface로 보고, 수정 중인 feature 결합 훅부터
`src/features/{aggregate}/{capability}/hooks`로 이동한다.

## Import 경계

`src/shared/ui/**`는 가장 좁은 import 경계를 가진다.

허용:

- React (단, `import React from 'react'` 네임스페이스 사용 금지 — named import만 허용.
  자세한 규칙은 `docs/standards/typescript-code-style.md` 참고)
- MUI와 MUI icons
- 도메인과 무관한 내부 helper 또는 `src/utils/shared`
- 같은 컴포넌트 디렉토리의 파일
- Storybook/test 파일에서만 test utility, fake data, testing library

금지:

- `src/domain/**`
- `src/api/**`, generated REST SDK
- `src/graphql/**`, GraphQL document, generated DTO
- `src/hooks/**`
- `src/pages/**`
- `src/store/**`, Recoil atom/selector
- route/path constant, permission, domain context
- `src/components/{domain}/**` 또는 `src/features/{aggregate}/{capability}/ui`

feature UI는 shared UI를 import할 수 있다. shared UI는 feature UI를 import할 수 없다.
이 방향을 어기면 shared UI가 더 이상 제품 공통 contract가 아니게 된다.

## 전환 매핑

| 현재 분류                | 목표                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| `common-ui`              | official 기준을 만족할 때 `src/shared/ui/{ComponentName}`로 승격                               |
| `common-pattern`         | 단순한 UI contract로 만들 수 있으면 `src/shared/ui/{ComponentName}`로 승격, 아니면 legacy 유지 |
| `common-template`        | page/layout 표준이 별도로 정의되기 전까지 legacy 유지                                          |
| `common-hook-or-util`    | 여러 feature에서 공유되면 `src/shared/hooks` 또는 `src/shared/utils`로 이동. feature 결합 hook은 `src/features/{aggregate}/{capability}/hooks`로 이동한다 |
| `feature-ui`             | feature 이관 시 `src/features/{aggregate}/{capability}/ui`로 이동                              |
| `domain-logic-candidate` | 도메인 경계가 명확할 때 `src/domain/{aggregate}/{capability}`로 이동                           |
| `legacy-defer`           | 기존 위치를 유지하고, 변경 전 focused migration plan을 작성                                    |

## URL State

URL params 처리는 전부 도메인 로직으로 보지 않는다.

- 공통 메커니즘 후보: `useUrlFilterState`, `useUrlTabState`, `useUrlPaginationState`
- 도메인 소유 로직: order filter parse/serialize, API input mapping, enum 변환

URL state의 일반 메커니즘은 shared hook 후보가 될 수 있다. 도메인 필드 해석은
도메인 경계에 둔다.

## 자동화 방향

최종 guard는 contributor의 기억 비용을 낮추는 방향이어야 한다. 수동 실행해야 하는
repo-local script보다 자동화된 check를 우선한다.

권장 순서:

1. 이 문서에 컨벤션을 확정한다.
2. 기존 컴포넌트 slice를 리뷰하고 커밋할 때 이 기준을 적용한다.
3. changed file 기준의 가벼운 custom lint rule 또는 lint-staged guard를
   먼저 추가한다.
4. false positive가 낮아진 뒤 CI로 확장한다.

초기 check는 확실한 위반만 막는다.

- `src/components/**` 아래 신규 official shared UI 추가
- `shared/common` 디렉토리 추가
- `src/shared/ui/{ComponentName}`에 Storybook story 누락
- `src/shared/ui/**`에서 domain import 사용
