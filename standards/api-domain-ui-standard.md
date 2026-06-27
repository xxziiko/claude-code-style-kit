# API / Domain / UI 표준

> 프로젝트 전반의 API 호출, domain 경계, UI 데이터 접근 방식을 일관되게 맞추기 위한 기준 문서

- Status: Baseline v1 (Agreed)
- Date: 2026-04-23
- Scope: Repo-wide

## 1. 목적

이 문서는 프로젝트 전반에서 API 호출, domain 경계, UI 데이터 접근 방식을 일관되게 맞추기 위한 기준 문서다.

핵심 목표는 아래 세 가지다.

1. 데이터 접근 경로를 예측 가능하게 만든다.
2. generated 타입과 UI 타입이 뒤섞이지 않게 한다.
3. REST 전환 이후에도 동일한 패턴으로 유지되게 만든다.

이 표준의 상위 지향점은 아래와 같다.

1. 프로젝트 구조와 타입 모델이 비즈니스 문서를 대체할 수 있을 정도로 명확해야 한다.
2. `domain`은 구현 세부사항이 아니라 비즈니스 언어를 드러내는 중심이어야 한다.
3. 테스트는 단순 회귀 방지가 아니라 비즈니스 정책의 실행 가능한 문서여야 한다.

## 2. 아키텍처 기준

이 프로젝트의 현재 표준 방향은 `domain-first`다.

의미:

1. `domain`은 비즈니스 경계의 source of truth다.
2. API 전환과 표준화는 `domain`을 중심으로 점진적으로 진행한다.
3. `pages`, `components`, `hooks`는 유지하되, 비즈니스 규칙과 데이터 접근 책임은 계속 `domain`으로 끌어올린다.
4. 지금 단계에서는 `feature-sliced` 전면 도입보다 `domain` 경계 명확화가 우선이다.
5. `domain` 분리 기준은 endpoint, transport, 화면이 아니라 비즈니스 개념이다.

### 2-1. Aggregate / Capability 모델링

도메인 이름은 먼저 큰 aggregate를 확인한 뒤, 필요할 때만 하위 capability로 나눈다.
endpoint action 이름을 그대로 새 최상위 domain으로 만들지 않는다.

현재 큰 aggregate 기준:

| Aggregate  | 범위                                                                       | 예                                                       |
| ---------- | -------------------------------------------------------------------------- | -------------------------------------------------------- |
| `order`    | OMS 주문, 주문 상태, 송장, 배송일, 작업 설정, 취소/롤백, 주문 기반 command | `order`, `orderCollect`, `invoice`                       |
| `wms`      | WMS 입출고/재고/패킹/로케이션/공급처/카테고리 등 창고 운영                 | `receiptHistory`, `packing`, `stockOverview`, `supplier` |
| `business` | 거래처/비즈니스 상품/주문 통계/프리셋 등 business-facing 영역              | 기존 GraphQL business hooks 및 화면                      |
| `admin`    | 관리자 계정, 권한, feature flag 등 운영자 설정                             | `featureFlag`, admin account                             |

하위 domain/capability 분리 기준:

1. 같은 aggregate 안에서도 사용자 업무 언어가 독립적이면 분리한다.
   - 예: `orderCollect`, `invoice`
2. query / history / command 성격이 명확히 다르면 분리할 수 있다.
   - 예: `receiptHistory`는 query, `packing`은 command
3. 단일 endpoint/action 이름만으로 새 domain을 만들지 않는다.
   - 지양: `orderCommand`, `updateWorkSettings`
   - 권장: 기존 `order`에 command로 추가하거나, 반복되는 독립 업무라면 `orderWorkSettings`처럼 capability 이름을 사용
4. `command`는 mode 또는 파일 역할이지 domain 이름이 아니다.
5. cancel/rollback처럼 id 의미나 업무 의미가 다르면 같은 aggregate 아래 별도 action으로 모델링하되, 무리하게 통합하지 않는다.

#### Commerce / WMS endpoint 경계 예외

백엔드 endpoint나 generated controller 이름에 `commerce`가 들어가더라도, 프론트 domain 이름을 `commerce*`로 만들지 않는다.
`commerce`는 API 출처 또는 외부 채널 이름일 수 있지만, 이 레포의 domain ownership은 사용자가 수행하는 업무 aggregate 기준으로 정한다.

WMS generated SDK를 호출하더라도 아래 업무는 기본적으로 `order` aggregate에 속한다.

| Operation / URL                                | 금지/지양 domain   | 권장 aggregate/capability  | 이유                                                                    |
| ---------------------------------------------- | ------------------ | -------------------------- | ----------------------------------------------------------------------- |
| `/api/v1/commerce/delivery/restriction-check*` | `commerceDelivery` | `orderDeliveryRestriction` | 출고/송장/패킹 전 배송 제한 정책 체크이며, 자사몰 전용 도메인이 아니다. |

기존 WMS domain과의 경계:

- `stockOverview` / `stockStatus`: 재고 조회 query. 주문 상품 매핑 API를 소유하지 않는다.
- `stockAllocation`: 재고 부족 주문의 대체 상품 매핑. 주문 aggregate command와 분리한다.
- `packing`: 패킹 작업 command. 송장번호 기반 출고제한 체크를 호출할 수는 있지만 정책 소유 domain은 `orderDeliveryRestriction`이다.
- `receipt*`, `ripening`, `supplier`: 입고/검수/숙성/공급처 기준정보이며 주문/배송 상태 모델링과 분리한다.

추천 결정 순서:

```txt
1. 이 API가 어떤 aggregate에 속하는가? order / wms / business / admin
2. 기존 domain에 같은 업무 언어가 있는가?
3. query, command, workflow 중 무엇인가?
4. 새 domain이 필요하다면 endpoint명이 아니라 업무 capability 이름인가?
5. 단일 action이면 기존 aggregate domain의 mutation으로 충분하지 않은가?
```

## 3. 표준 경로

신규/전환 코드는 아래 경로를 기본으로 따른다.

```txt
OpenAPI generated SDK
  -> domain/api.ts
  -> domain/mappers|factories.ts
  -> domain/queries.ts | domain/mutations.ts
  -> page/component
```

UI는 가능하면 domain 모델만 사용한다.

## 4. 구조 원칙

현재 디렉토리 구조를 유지하더라도, 새 코드와 전환 코드는 아래 원칙을 따른다.

```txt
src/
  api/           -> generated SDK (수동 수정 금지)
  domain/        -> 비즈니스 경계, 데이터 접근, 타입, 규칙
  features/      -> UI, feature hook, feature-local display/interaction types
  pages/         -> 화면 조합과 라우팅 진입점
  shared/        -> 공용 UI/hook/constant/util (ui/, hooks/, constants/, utils/)
  components/    -> legacy 표현/재사용 UI (점진적으로 shared/ui 또는 features/{x}/{y}/ui 로 이관)
  hooks/         -> legacy hook (점진적으로 shared/hooks 또는 features/{x}/{y}/hooks 로 이관)
```

핵심은 폴더 이름이 아니라 책임 경계다.

### `domain`이 소유하는 것

- API 호출
- request / response 변환
- frontend domain language and business types
- query key
- mutation / invalidate 전략
- 비즈니스 규칙

### `features/{aggregate}/{capability}/hooks`가 소유하는 것

- search params
- selection
- drawer / modal open state
- pagination / filter state
- 여러 domain query/mutation의 UI orchestration
- 도메인 UI나 도메인 하위 화면에 강하게 결합된 local UI state

### `components`가 소유하는 것

- 표현
- 상호작용 UI
- domain 결과를 렌더링하는 화면 조각

### `pages`가 소유하는 것

- 라우팅 진입점
- 화면 조합
- page-level composition

## 5. 레이어별 책임

### `src/api/generated/*`

- OpenAPI 기반 자동 생성 코드
- 수동 수정 금지
- SDK / DTO / generated type source

### `src/domain/{aggregate}/{capability}/api.ts`

- generated SDK 호출 진입점
- request DTO 조립
- response envelope 처리 (success/data/error 분기)
- error 정규화와 throw

`api.ts`는 DTO -> Domain 변환을 직접 소유하지 않는다. 모든 변환은 `mappers.ts`
또는 `factories.ts`에 위임하고 `api.ts`는 mapper import 후 호출만 한다.

### `src/domain/{aggregate}/{capability}/types.ts` and `dto.ts`

- `types.ts`: capability가 제공하는 업무 데이터 계약과 API 호출 경계 params/payload/result
- `dto.ts`: generated-adjacent DTO/envelope alias
- generated type 재노출 금지
- UI interaction/display 타입은 feature layer `types.ts`에 둔다

#### REST Type Naming Decision Table

| 의미                        | 이름                                           | 위치                            | 예                          |
| --------------------------- | ---------------------------------------------- | ------------------------------- | --------------------------- |
| REST response display model | `{Domain}Item`                                 | `types.ts`           | `SupplierItem`              |
| REST list wrapper           | `{Domain}List`                                 | `types.ts`           | `SupplierList`              |
| REST list/detail params     | `{Domain}ListParams`, `{Domain}DetailParams`   | `types.ts`                  | `SupplierListParams`        |
| REST command params         | `Create{Domain}Params`, `Update{Domain}Params` | `types.ts`                  | `CreateSupplierParams`      |
| Validated domain entity     | `{Domain}Entity`                               | concrete business file          | `SupplierEntity`            |
| Validated entity input      | `Create{Domain}EntityInput`                    | concrete business file          | `CreateSupplierEntityInput` |
| UI table row view model     | `{Domain}TableRow`                             | feature `types.ts`              | `SupplierTableRow`          |

원칙:

- `Supplier`처럼 단독 domain명만 쓰는 타입은 새 REST 모델에서 피한다. 화면 컴포넌트, REST record, 검증 entity 의미가 쉽게 충돌한다.
- REST caller가 API 응답 display model 그대로 쓰면 `{Domain}Item`을 직접 import한다.
- `export type SupplierListItem = SupplierItem` 같은 동일 shape alias는 만들지 않는다.
- table 전용 파생 필드가 생길 때만 `{Domain}TableRow`를 만든다.
- smart constructor를 품는 검증 완료 객체가 필요할 때만 `{Domain}Entity`를 만든다.
- hook 파일은 타입 공개 API가 아니다. hook return type처럼 hook 자체의 API가 아닌 공용 타입은 domain public surface 또는 feature `types.ts`에 둔다.

### `src/domain/{aggregate}/{capability}/queries.ts`

- query key factory
- `queryOptions()` 정의
- 캐시 전략(staleTime/gcTime) 정의

### `src/domain/{aggregate}/{capability}/mutations.ts`

- `mutationOptions` factory 정의
- domain 공통 mutation 동작 정의
- invalidate / refetch 전략 정의

### `src/domain/{aggregate}/{capability}/mappers.ts` or `factories.ts`

- 모든 DTO -> Domain 변환 함수의 단일 소유 위치
- 단순 단일 필드 rename mapper도 예외 없이 이 파일이 소유한다
- 복잡한 DTO -> domain mapping
- 파생 값 조립
- `mappers.ts`는 변환 함수, `factories.ts`는 domain construction/aggregate 조립

단순/복잡을 가르는 threshold는 두지 않는다. `api.ts`에는 변환 함수를 남기지 않는다.

### hook 배치

- feature-specific hook: `src/features/{aggregate}/{capability}/hooks/`에 둔다.
  예: `src/features/stock/expiration/hooks/useStockExpirationDrawer.ts`
- 도메인 무관 공용 hook: `src/shared/hooks/`에 둔다.
- 기존 `src/hooks/**`는 legacy surface다. 신규 hook을 그 아래에 추가하지 않는다.
  수정 중인 hook은 위 두 target 중 한 곳으로 이동한다.
- 얇은 query wrapper hook 금지. `useSuspenseQuery(xxxQueries.list(...))`만 감싸지 않는다.

### `src/components/**`, `src/pages/**`

- domain query/mutation 사용
- SDK 직접 호출 금지
- direct `fetch` 금지

## 6. 허용 / 금지 규칙

### 허용

- page/component -> `domain/queries.ts`, `domain/mutations.ts` 직접 import
- page/component -> domain hook 또는 shared hook import
- domain/api -> generated SDK import
- domain 내부 mapper/factory 분리

### 금지

- UI에서 `@api/generated/*` 직접 import
- UI에서 direct `fetch`
- UI에서 generated DTO 타입 직접 사용
- query key 문자열 하드코딩
- queryKey / queryFn 중복 정의
- `useSuspenseQuery(xxxQueries.list(...))`만 감싸는 단순 wrapper hook
- domain 공개 API를 barrel export로 재노출
- UI 레이어에 비즈니스 규칙을 새로 추가하는 것
- `components`나 `pages` 아래에 domain 역할을 하는 로직을 누적시키는 것
- `src/hooks/{domain}` 아래에 신규 feature/domain 결합 훅을 추가하는 것

## 7. 점진 마이그레이션 원칙

이 표준은 한 번에 전체 구조를 뒤엎는 방식이 아니라, 아래 방식으로 적용한다.

1. 신규 코드는 무조건 `domain` 경유로 작성한다.
2. 기존 코드는 수정하거나 전환할 때마다 `domain` 경계로 흡수한다.
3. `pages/components/hooks`를 먼저 지우지 않는다.
4. 대신 business logic과 data access를 점진적으로 `domain`으로 이동시킨다.
5. hook 배치 규칙은 기존 코드에도 동일한 판정 기준으로 적용한다.
6. 다만 기존 코드 전체를 한 번에 리팩토링하지 않고, 수정/전환 시 우선순위에 따라 `src/features/{aggregate}/{capability}/hooks` 또는 `src/shared/hooks`로 흡수한다.

## 8. 기본 폴더 구조

```txt
src/domain/{aggregate}/{capability}/
  index.ts
  api.ts
  types.ts
  dto.ts              # 필요 시
  queries.ts
  mutations.ts        # 필요 시
  mappers.ts          # 필요 시
  factories.ts        # 필요 시
  rules.ts            # 필요 시
  __tests__/
```

원칙:

- 응집도가 높으면 같은 파일 유지
- 변환/규칙/생성이 커질 때만 분리
- 파일 분리는 재사용성과 복잡도를 기준으로 한다
- DTO -> Domain 변환은 단순 여부와 무관하게 `mappers.ts` 또는 `factories.ts`가 소유한다.
- 도메인 값 생성 경계가 중요하면 `expirationDate.ts`, `stockQuantity.ts`, `supplierCode.ts`처럼 구체적인 비즈니스 이름의 파일로 분리한다.
- `src/domain/shared/**/index.ts` barrel도 만들지 않는다. shared-kernel 예외는 두지 않으며, 호출처는 deep path로 직접 import한다.

#### 도메인 횡단 공통 모듈

도메인이 반복적으로 같은 query/value-object/result item을 사용하면 공통 모듈을 둘 수 있다.
이때 위치는 `src/domain/shared`가 아니라 의미상 가장 가까운 aggregate 아래에 둔다.

예: 주문 command 결과에 반복되는 allocation result item

- 위치: `src/domain/order/allocationResult/{types,mappers}.ts`
- public domain surface로 인정되며, `invoice` 같은 sibling capability도 재사용할 수 있다.
- aggregate 외부에서 import할 때는 deep path를 사용한다 (barrel 금지).

`src/domain/stockAllocation`의 mapper/factory/type은 별도 README로 문서화된
public query surface 예외다. UI orchestration hook에서 deep path로 직접
import해도 된다. 새 예외를 추가할 때는 같은 방식으로 문서화한다.

## 9. 타입 경계 규칙

1. generated type은 domain 내부까지만 허용한다.
2. UI에 전달되는 타입은 domain type이어야 한다.
3. generated DTO를 그대로 `extends` 하거나 `as` 캐스팅으로 밀어 넣지 않는다.
4. request body도 필요하면 domain input type으로 재정의한다.

### 9-1. 레이어별 타입 배치 기준

도메인 내부에서 타입을 생성하기 전에 아래 질문을 순서대로 적용한다.
신규 scaffold는 `src/domain/{aggregate}/{capability}/types.ts`와 필요 시 `dto.ts`
구조를 기본으로 한다. `types.ts`는 capability가 노출하는 업무 데이터 계약과
API 호출 경계 타입을 함께 담고, raw DTO는 `dto.ts`, UI 상태는 feature `types.ts`,
행위가 있는 업무 개념은 승인된 manifest의 concrete business file에 둔다.

#### 타입 생성 전 의사결정표

| 질문                                                                                                   | 위치                                                      | 예                                              |
| ------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- | ----------------------------------------------- |
| PRD, 정책 문서, wiki/Notion 등에서 정의된 aggregate의 업무 정책/불변식/기준정보인가?                   | concrete business file                                    | `src/domain/stock/stockQuantity.ts`             |
| 특정 하위 domain/capability의 조회 결과나 query인가?                                              | `src/domain/{aggregate}/{capability}/types.ts` | `src/domain/stock/overview/types.ts` |
| `api.ts`, `queries.ts`, `mutations.ts` 호출 경계의 params/payload인가?                                 | `src/domain/{aggregate}/{capability}/types.ts`        | `src/domain/stock/overview/types.ts`        |
| 서버 raw response/request, generated 타입 alias, envelope shape인가?                                   | `src/domain/{aggregate}/{capability}/dto.ts`        | `src/domain/stock/overview/dto.ts`        |
| tab, drawer, selection, form draft, display row, footer row, component state인가?                      | `src/features/{aggregate}/{capability}/types.ts`          | `src/features/stock/overview/types.ts`          |

스캐폴더와 에이전트는 승인된 manifest에 없는 파일을 신규 생성하지 않는다.
추가 파일이 필요하면 먼저 manifest를 갱신하고 검토한다.

분리된 타입 파일은 파일 상단 JSDoc으로 레이어 책임을 명시한다. 특히
capability `types.ts`는 순수 DDD entity가 아니라 해당 capability가 소유하는
업무 데이터 계약/query임을 적고, UI interaction/display type은 feature `types.ts`에
둔다는 경계를 함께 적는다.

예:

```ts
/**
 * Stock overview capability query types.
 *
 * 이 파일은 stock aggregate의 순수 DDD entity가 아니라, stock overview 조회
 * capability가 UI 표현/상호작용과 독립적으로 제공하는 업무 데이터 계약을 정의한다.
 *
 * Aggregate 공통 업무 개념은 `branded.ts` 또는 구체적인 business filename,
 * API params/payload는 `types.ts`, raw DTO는 `dto.ts`,
 * UI interaction/display state는 `src/features/stock/overview/types.ts`에 둔다.
 */
```

#### Concrete business file과 capability query

`types.ts`는 capability 조회 계약과 API 호출 경계 타입의 위치다. PRD/정책 문서/wiki/Notion 등에서
정의된 aggregate 업무 정책, 기준정보, 불변식은 구체적인 business filename으로 둔다.

1. Aggregate 공통 business concept

   - 위치: `src/domain/{aggregate}/{businessName}.ts` 또는 aggregate-wide branded primitive는 `src/domain/{aggregate}/branded.ts`
   - PRD, 정책 문서, wiki/Notion, 업무 회의록에서 명시된 기준정보, 정책 분류, 불변식, 값 객체를 코드로 표현한다.
   - UI, API endpoint, query cache와 무관해야 하며, 정책 코드, type guard, smart constructor의 문서화 대상이다.
   - 예: `stockQuantity.ts`, `expirationDate.ts`, `dateRange.ts`, `allocationResult.ts`
   - 단순히 여러 화면에서 재사용된다는 이유만으로 aggregate business file로 승격하지 않는다. 명확한 업무 개념/정책이 있을 때만 만든다.

2. Capability query
   - 위치: `src/domain/{aggregate}/{capability}/types.ts`
   - 특정 조회/화면 capability가 소유하는 업무 데이터 계약/query이다.
   - 순수 DDD entity는 아니지만, DTO나 UI 상태가 아니라 UI 표현/상호작용과 독립된
     domain-facing output contract다.
   - 예: `StockOverviewResult`, `StockOverviewItem`, `StockExportItemList`

신규 파일은 승인된 manifest에 포함된 경우에만 만든다.

#### `types.ts`: API 호출 경계 타입

`api.ts`는 domain adapter 함수와 query/mutation factory가 공유하는 호출 경계 타입을
소유한다.

허용:

- `FetchStockOverviewParams`
- `FetchStockExportItemsParams`
- `PostStockExportParams`
- `UpdateOrderStockItemParams`

규칙:

- 조회 params와 mutation payload는 UI selection/form state를 그대로 사용하지 않는다.
- UI가 가진 filter/selection state는 feature hook에서 API params/payload로 변환한다.
- `types.ts`는 concrete business file 또는 capability query type을 import할 수 있다.
- `types.ts`는 UI type을 import하지 않는다.
- `types.ts`를 자유롭게 import할 수 있는 곳은 `api.ts`, `queries.ts`, `mutations.ts` 세 곳뿐이다. feature hook은 UI state를 Params로 변환하는 explicit conversion boundary가 될 수 있지만, Params 타입은 `@domain/{aggregate}/{capability}` public surface를 통해서만 받는다. `@domain/*/types` deep import는 feature hook에서도 금지하며, `pages/*`, `components/*`, `shared/*`에서는 직접 import하지 않는다.
- 세부 사용 권한은 아래 ``dto``와 ``types`` 호출 경계 규칙 (결정적) 표를 따른다.

#### `dto.ts`: 외부 API raw shape

`dto.ts`는 REST/OpenAPI/generated/legacy endpoint가 반환하거나 요구하는 raw shape를
소유한다.

허용:

- `StockOverviewResponseDto`
- `StockOverviewItemDto`
- `StockExportItemsResponseDto`
- generated 타입 alias: `AllStockItemListResponse as StockOverviewListDto`

규칙:

- DTO 타입은 `Dto` suffix를 사용한다.
- `dto.ts`는 `api.ts`와 `mappers.ts`/`factories.ts`만 import한다.
- UI, hooks, queries, mutations는 DTO를 직접 import하지 않는다.
- DTO -> query/domain type 변환은 `mappers.ts` 또는 `factories.ts`가 소유한다.
- `dto.ts`를 import할 수 있는 곳은 `api.ts`와 `mappers.ts`/`factories.ts` 둘 뿐이다. `queries.ts`, `mutations.ts`, `features/*`, `pages/*`, `components/*`, `shared/*`는 예외 없이 금지다.
- mapper를 거친 이후에는 DTO가 domain 어디에도 다시 등장하지 않는다.
- 세부 사용 권한은 아래 ``dto``와 ``types`` 호출 경계 규칙 (결정적) 표를 따른다.

#### Feature `types.ts`: UI 상태와 표현 타입

UI type은 React/component interaction과 화면 표시를 위해 생기는 타입이다.

허용:

- `StockQuantityTab`
- `ActiveStockDrawer`
- `StockExportOption`
- `StockExportSelection`
- `StockItemDetailDisplayRow`
- `StockUnitDisplayRow`

규칙:

- `Tab`, `Drawer`, `Selection`, `Option`, `DisplayRow`, `ViewMode`, `Draft`,
  `FooterRow`는 UI type 후보로 먼저 본다.
- `isFooter`, `actionable`, `displayLabel`, `checked`, `selected`, `expanded`처럼
  렌더링이나 interaction을 위해 추가된 필드는 UI type에 둔다.
- UI type은 `src/features/{aggregate}/{capability}/types.ts`에 둔다.
- UI type은 필요한 경우 concrete business file 또는 capability query type을 import할 수 있다.
- UI type은 DTO를 import하지 않는다.
- `api.ts`, `queries.ts`, `mutations.ts`, `mappers.ts`는 UI type을 import하지 않는다.

#### Business concept / query / UI의 구분 기준

어떤 타입이 concrete business concept, capability query, UI 중 어디에 속하는지 애매하면 아래 질문으로 결정한다.

| 질문                                                                                                | 예이면                           |
| --------------------------------------------------------------------------------------------------- | -------------------------------- |
| PRD, 정책 문서, wiki/Notion 등에서 aggregate의 업무 정책/기준정보/불변식으로 정의된 개념인가?       | concrete business file           |
| 단순 조회 결과가 아니라 정책 코드, type guard, smart constructor로 문서화할 대상인가?               | concrete business file           |
| 특정 capability가 UI 표현/상호작용과 독립적으로 제공하는 업무 데이터 계약인가?                      | capability `types.ts` |
| 탭, drawer, 선택, 입력 draft, footer, 렌더링 분기를 표현하는가?                                     | UI                               |
| 필드가 layout/rendering/interactions 때문에 생겼는가?                                               | UI                               |
| API request/response shape와 1:1로 맞추기 위해 생겼는가?                                            | dto 또는 api                     |

예를 들어 `StockOverviewItem`은 테이블 렌더링 방식과 무관하게 재고 현황 조회
capability가 제공하는 업무 데이터 계약이므로 `stock/overview/types.ts`에 둔다.
반면 `StockItemDetailDisplayRow`는 footer,
actionable cell, display label처럼 테이블 렌더링을 위해 재구성된 shape이므로
feature `types.ts`에 둔다.

#### 의존성 방향

```txt
concrete business file
  <- capability types
  <- feature types.ts

capability dto
  -> api.ts / mappers.ts only

capability types
  -> api.ts / queries.ts / mutations.ts
  -X feature types.ts

domain api.ts / queries.ts / mutations.ts
  -> concrete business file
  -> capability types
  -> types
  -X feature types.ts
  -X dto except api.ts/mappers.ts boundary

feature ui/hooks
  -> concrete business file
  -> capability public surface
  -> feature types.ts
  -> Params only through domain public surface at explicit conversion boundary when unavoidable
  -X domain internal types.ts deep import
  -X dto
```

`domain/api.ts`, `domain/queries.ts`, `domain/mutations.ts`에서 feature `types.ts`를 import하는 것은
경계 위반이다. feature type이 필요해 보이면 해당 타입이 실제로는 concrete business concept, query 또는 api 타입인지 먼저 재분류한다. 이 경계는 커스텀 ESLint 규칙 `sirloin/domain-type-boundary`로도 검사한다.

#### `dto`와 `types` 호출 경계 규칙 (결정적)

`dto`와 `types` 둘 다 **도메인 내부 경계 타입**이다. UI 컴포넌트가 이들을 직접 import하면 layered 경계가 무너진다. 두 타입의 차이는 "UI가 쓸 수 있느냐"가 아니라 **어느 도메인 내부 경계까지 살아남느냐**이다.

이 규칙은 의견이 아니라 결정 사항이다. 구현, 리뷰, 자동 agent (Codex/Claude) 작업은 다른 어떤 해석보다 본 표와 흐름도를 우선 근거로 삼는다.

##### 사용 권한 표

| 타입군 | 위치 | 대표 suffix | 자유롭게 import 가능 | 제한적 import 가능 | 절대 import 불가 |
| --- | --- | --- | --- | --- | --- |
| Dto (wire raw) | `dto.ts` | `XxxResponseDto`, `XxxRequestDto`, `XxxResultDto`, `XxxResponseItemDto` | `api.ts`, `mappers.ts`/`factories.ts` | (없음) | `queries.ts`, `mutations.ts`, `features/*`, `pages/*`, `components/*`, `shared/*` |
| Params (호출 경계 input) | `types.ts` | `XxxParams`, `XxxPayload` | `api.ts`, `queries.ts`, `mutations.ts` | `features/{aggregate}/{capability}/hooks/*` — `@domain/{aggregate}/{capability}` public surface를 통해 export된 타입만, explicit conversion boundary에 한해 unavoidable일 때만 | `features/{aggregate}/{capability}/ui/*`, `pages/*`, `components/*`, `shared/*`, `@domain/*/types` deep import from feature/UI |

##### 단방향 데이터 흐름

```txt
[UI 컴포넌트]
   selection / filter / form draft (feature types.ts)
        ↓
[features/{aggregate}/{capability}/hooks/*]
   ── conversion boundary ──   (UI state → Params, domain public surface를 통해 이 한 곳에서만 Params에 닿는다)
        ↓
[queries.ts / mutations.ts]
   Params 자유 사용 → factory가 api.ts 호출
        ↓
[api.ts]
   Params → URLSearchParams 또는 RequestDto
        ↓ HTTP ↑
   ResponseDto (wire raw, Dto의 마지막 등장)
        ↓
[mappers.ts]
   ResponseDto → Result (query)
        ↑
   (이 지점 이후로 Dto는 domain 어디에도 다시 등장하지 않는다)
```

##### 자가 검증 체크리스트

다음 grep 패턴이 적중되면 위반이므로 머지 전 교정한다.

1. `pages/**`, `components/**`, `shared/**` 어느 곳에서든 `from '@domain/*/dto'` → 위반.
2. `pages/**`, `components/**`, `shared/**` 어느 곳에서든 `from '@domain/*/types'` → 위반.
3. `src/features/**`에서 `from '@domain/*/dto'` → 항상 위반.
4. `src/features/**`에서 `from '@domain/*/types'` → 항상 위반. Params가 필요하면 `@domain/{aggregate}/{capability}` public surface에서 import하고 conversion boundary를 feature hook 한 곳으로 모은다.
5. `src/domain/*/ui/**` 또는 `src/domain/*/*/ui/**` 신규 파일 → 위반. UI는 feature layer에 둔다.
6. `src/domain/*/queries.ts` 또는 `mutations.ts`에서 `from '@domain/*/dto'` → 위반. queries/mutations는 Params만 다루고 Dto는 보지 않는다.

##### 흔한 위반 패턴과 교정

| 위반 | 교정 |
| --- | --- |
| 페이지가 `StockItemDetailParams`를 import해 객체를 조립한 뒤 `useQuery(stockQueries.itemDetail(params))` 에 전달 | 페이지는 자신의 UI state (`stockItemId`, `groupBy` 같은 string/UI value) 만 보유. domain orchestration hook (예: `useStockItemDetailQuery(stockItemId, groupBy)`) 이 해당 state를 받아 내부에서 Params로 조립하고 `queryOptions`를 떨궄다. |
| 컴포넌트가 `StockOverviewResponseDto`를 import해 응답 일부 필드를 직접 사용 | 컴포넌트는 mapper를 통과한 query (예: `StockOverviewResult`, `StockOverviewItem`) 만 사용. mapper에 누락된 필드가 있으면 mapper를 보강하고 DTO를 노출하지 않는다. |
| domain hook이 `Params`를 인자로 받아 그대로 `queries.ts` factory에 전달 | hook 인자는 UI state (예: `{ stockItemId, groupBy }`) 로 받고, hook 내부에서 Params로 조립한다. hook 자체가 conversion boundary다. |

##### 결정적 강제

- 추가 혹은 리뷰 중 자가 검증 체크리스트에 해당하는 import가 발견되면 머지 전 즉시 교정한다.
- ESLint `sirloin/domain-type-boundary` 규칙이 자동 강제 범위를 단계적으로 확대하며, 그 이전에는 리뷰와 grep 체크리스트가 강제 장치다.
- agent (Codex/Claude) 작업 시 DTO 또는 Params 사용 권한이 헷갈리면 본 사용 권한 표를 먼저 근거로 삼고, 모호하면 작업을 멈추고 결정을 요청한다.

## 10. Smart Constructor / Business Rule File 규칙

비즈니스적으로 유효한 값만 존재해야 하는 타입은 smart constructor를 둔다.

예:

```txt
src/domain/{aggregate}/{capability}/{businessName}.ts
```

원칙:

1. `as SomeDomainType` 캐스팅으로 유효성을 우회하지 않는다.
2. `create{ValueName}` 형태의 smart constructor를 둔다.
3. 필요하면 `is{ValueName}` type guard를 둔다.
4. 실패는 domain error로 표현한다.
5. 값 객체가 public contract의 일부라면 `index.ts`에서 의도적으로 공개하거나, domain 내부에서만 사용한다.
6. 타입 파일은 다른 파일의 타입/함수를 passthrough re-export하지 않는다.
7. domain 내부 구현과 테스트는 구체적인 business rule 파일을 직접 import할 수 있다.

예시:

```txt
src/domain/{aggregate}/{capability}/{businessName}.ts
```

- `create{ValueName}(value: string): {ValueName}`
- `is{ValueName}(value: string): value is {ValueName}`
- `Unsupported{ValueName}Error`

### Generated Runtime Contract 패턴

generated 타입이 string union만 제공하고 runtime 값 목록은 제공하지 않을 때,
다음 패턴을 사용한다.

```txt
generated types
  -> post-codegen runtime contracts
  -> domain smart constructor
```

예:

```txt
src/api/generated/{service}/types.gen.ts
  -> scripts/generate-runtime-contracts.mjs
  -> src/api/generated/{service}/contracts.gen.ts
  -> src/domain/{aggregate}/{capability}/{businessName}.ts
```

원칙:

1. generated는 계약의 source of truth다.
2. runtime contracts는 generated에서 파생된 런타임 상수다.
3. domain value object는 runtime contracts를 소비해 smart constructor를 제공한다.
4. 수동 배열을 별도 source of truth로 두지 않는다.

### codegen 연결 원칙

REST codegen을 다시 돌리면 runtime contracts도 함께 재생성되어야 한다.

현재 기준:

```txt
yarn codegen:rest
  -> openapi-ts
  -> yarn codegen:contracts
```

즉, `codegen:rest`는 generated types/sdk만 갱신하는 명령이 아니라
runtime contracts까지 포함한 전체 REST 계약 갱신 명령으로 본다.

### 언제 generated runtime contracts를 만들 것인가

신규 API 타입이 아래 조건을 만족하면 runtime contract 생성 후보로 본다.

1. generated가 string union 또는 enum-like type을 제공한다.
2. 그 값을 런타임에서 검증해야 한다.
3. domain에서 smart constructor 또는 type guard를 만들고 싶다.
4. UI/도메인 전반에서 반복적으로 참조될 가능성이 있다.
5. 수동 배열을 별도 source of truth로 두면 drift 위험이 있다.

### 적합한 예

- action 목록
- status 값
- reason code
- delivery type
- category code
- mode / phase / kind 처럼 허용값 집합이 중요한 값

### 적합하지 않은 예

- 단순 문자열 필드
- 한 화면에서만 임시로 쓰는 상수
- 런타임 검증이 필요 없는 값
- domain이 별도 value object로 다루지 않는 값

### 신규 API 설계 시 적용 순서

1. generated 타입에서 union/enum-like 값을 확인한다.
2. 이 값이 domain value로 의미가 있는지 판단한다.
3. 의미가 있다면 `contracts.gen.ts` 생성 대상에 추가한다.
4. domain의 구체적인 business rule 파일에서 smart constructor를 제공한다.
5. 호출처는 domain public type만 사용한다.

예:

```txt
generated:
  SomeStatus = 'A' | 'B' | 'C'

contracts:
  SOME_STATUS_VALUES

domain/business-rule-file:
  createSomeStatus()
  isSomeStatus()
  UnsupportedSomeStatusError
```

## 11. Query / Mutation 규칙

### Query

- query key는 factory로 정의한다.
- `queryOptions()` 안에 `queryKey`, `queryFn`을 함께 둔다.
- staleTime/gcTime은 domain 성격에 맞게 명시한다.

### Mutation

- `mutations.ts`는 기본적으로 `useXxxMutation()` 커스텀 훅 대신 `mutationOptions` factory를 export 한다.
- mutation 이름은 동작 의도가 드러나야 한다.
- domain mutation은 기본적으로 `mutationFn`만 소유한다.
- `mutationKey`는 mutation identity를 실제로 소비할 때만 선택적으로 둔다.
- invalidate/refetch 전략은 호출부 또는 UI orchestration hook에서 정의한다.
- UI에서 query key를 직접 하드코딩해 invalidate하지 않는다.

### Handler vs Mutation 기준

서버로 가는 비동기 command는 그냥 핸들러에 숨기지 않고 mutation으로 상태를 관리한다.

원칙:

1. **핸들러는 트리거다.**
   - 버튼 클릭, 제출, 확인 같은 사용자 액션 진입점만 담당한다.
2. **mutation은 상태 관리자다.**
   - pending
   - success / error
   - mutate 호출
   - 후처리 연결 지점
3. 순수 UI 로컬 동작은 일반 핸들러로 충분하다.
4. 서버로 가는 비동기 command는 mutation으로 관리한다.

예:

```ts
const commandMutation = useMutation(domainMutations.execute());

const handleExecute = () => {
  commandMutation.mutate(payload, {
    onSuccess: () => {
      // 화면 후처리
    },
  });
};
```

즉, `핸들러냐 mutation이냐`가 아니라:

- 핸들러 = 언제 실행할지
- mutation = 실행 상태를 어떻게 관리할지

로 역할을 나눈다.

### 호출부 mutation 네이밍

호출부에서는 `useMutation()`의 결과 객체를 `{businessAction}Mutation` 이름으로
유지한다. mutation은 실행 함수 하나가 아니라 `mutate`, `isPending`,
`isError`, `error`, `isSuccess`, `reset`을 함께 가진 서버 command 상태
객체이기 때문이다.

권장:

```ts
const cancelOrderMutation = useMutation(orderCancelMutations.cancelOrder());

cancelOrderMutation.mutate(payload, {
  onSuccess: () => {
    // 화면 후처리
  },
});

if (cancelOrderMutation.isPending) {
  // loading UI
}
```

지양:

```ts
const { mutate: cancelOrder } = useMutation(orderCancelMutations.cancelOrder());

cancelOrder(payload);
```

`mutate` alias는 호출 함수처럼 보이지만 loading/error/success 상태와 분리되어
읽히므로 새 코드에서는 기본 패턴으로 쓰지 않는다. 렌더링 가독성을 위해
`const { isPending } = cancelOrderMutation`처럼 가까운 곳에서 상태만 구조분해하는
정도는 허용한다.

권장 형태:

```ts
import { mutationOptions } from '@tanstack/react-query';

export const domainMutations = {
  execute: () =>
    mutationOptions({
      mutationFn: (payload: DomainCommandPayload) =>
        executeDomainCommand(payload),
    }),
};
```

`mutationKey`가 필요한 경우는 `useMutationState`, `isMutating`, mutation defaults,
공통 pending 추적처럼 mutation identity를 실제로 소비할 때다.
단순 command 실행과 invalidate에는 필요하지 않다.

UI 사용 예시:

```ts
const actionMutation = useMutation({
  ...domainMutations.execute(),
  onSuccess: () => {
    // 화면 문맥 후처리
  },
});
```

### Mutation 옵션 관리 원칙

아래 두 방식은 모두 유효하다.

1. `useMutation({ ...domainMutations.execute(), onSuccess })`
2. `mutate(variables, { onSuccess, onError, onSettled })`

즉, 현재처럼 `mutate`의 두 번째 인자로 `onSuccess` 등을 넘기는 패턴은 여전히 유효하다.

권장 기준은 아래와 같다.

#### domain에 둘 것

- mutationFn
- 필요한 경우에만 domain 공통 mutationKey

#### 호출부 또는 UI orchestration hook에서 붙일 것

- alert / toast
- modal / drawer close
- refocus / navigation
- 화면 맥락에서만 필요한 후처리
- invalidateQueries / refetch
- 사용자 액션 핸들러에서 `mutate`를 호출하는 연결

호출부에서 query invalidation을 수행할 때는 domain key factory를 사용한다.
`queryKey: ['supplier']` 같은 문자열 하드코딩은 금지한다.

### 커스텀 mutation hook 허용 기준

domain `mutations.ts`에서는 단순 `useXxxMutation()` compatibility hook을 만들지 않는다.

반복되는 후처리가 필요하면 domain `mutations.ts`가 아니라 도메인 하위 UI/application
orchestration hook으로 분리한다.

예:

```ts
export function useDomainActionInScreen() {
  const queryClient = useQueryClient();

  return useMutation({
    ...domainMutations.execute(),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: domainKeys.lists() });
    },
  });
}
```

이 hook은 `domain/packing/mutations.ts`가 아니라 `src/features/wms/packing/hooks`처럼 해당
feature 하위 orchestration 계층에 둔다. 여러 feature에서 공유되는 순수 UI 상태 훅만
`src/shared/hooks`에 둔다.

## 11. Error 규칙

- SDK 에러는 domain/api에서 메시지를 정리한다.
- UI는 가능한 한 domain이 던지는 에러 메시지나 표준 에러 객체를 사용한다.
- error shape이 들쭉날쭉하면 domain에서 normalize 한다.

## 12. Error 예측 가능성 원칙

같은 domain 안에서는 에러 처리 방식도 최대한 일관되게 맞춘다.

1. 같은 종류의 실패는 같은 shape 또는 같은 메시지 전략을 따른다.
2. transport 세부사항은 가능하면 domain 밖으로 노출하지 않는다.
3. UI는 domain이 보정한 결과만 소비한다.

## 13. 캐시 / 실시간성 원칙

- 실시간성이 높은 목록/상세: 짧은 staleTime
- 기준정보/마스터 데이터: 긴 staleTime
- 기본값은 `src/index.tsx` QueryClient 설정을 참고하되, domain 특성에 맞게 조정한다.

## 14. 테스트 규칙

신규/전환 domain은 아래 중 필요한 검증을 포함한다.

- mapper / factory 단위 테스트
- value object / smart constructor 단위 테스트
- 호출부 또는 UI orchestration 테스트
- MSW 기반 유스케이스 테스트

추가 원칙:

0. 사용자 UI/UX에 영향을 줄 수 있는 변경은 관련 회귀 테스트가 통과해야만 완료로 본다.
1. 테스트 이름은 비즈니스 정책이나 규칙이 읽히도록 작성한다.
2. 테스트는 "어떤 구현을 썼는가"보다 "무슨 정책을 보장하는가"를 드러내야 한다.
3. domain 테스트는 가능하면 비즈니스 언어 기준으로 읽혀야 한다.
4. 라이브러리, generated SDK, React Query, MSW 자체의 동작을 검증하는 테스트는 작성하지 않는다.
5. transport wrapper만 검증하는 낮은 가치의 `api.test`는 작성하지 않는다.
6. `api.test.ts` 파일명은 금지한다. 의미 있는 검증은 `mappers.test.ts`,
   `factories.test.ts`, `failurePolicy.test.ts`, `rules.test.ts`처럼 정책/변환/
   도메인 규칙 이름의 테스트로 먼저 이동한 뒤 원본 `api.test.ts`를 삭제한다.
   단순 rename-only는 금지한다.
7. DTO -> domain 변환은 mapper/factory 테스트로 검증한다.
8. 에러/실패 정책은 `failurePolicy`, `rules`, `errors`처럼 정책이 드러나는 이름의 테스트로 검증한다.
9. mutation 이후 UI 갱신/refetch/alert 같은 정책은 API 테스트가 아니라 호출부 또는 orchestration 테스트로 검증한다.
10. MSW는 behavior regression에는 유효하지만, 실제 API 연결 안정성의 최종 증거로는 충분하지 않다.

회귀 테스트 적용 원칙:

- UI 표시/상호작용 변화 -> component 또는 hook integration 테스트 우선
- mutation 이후 갱신/refetch/alert/navigation 변화 -> orchestration 테스트 우선
- 여러 화면을 가로지르는 핵심 사용자 흐름 변화 -> E2E 또는 시나리오 테스트 우선
- 기존 회귀 테스트가 없으면 새로 추가하는 것을 기본값으로 한다.
- 자동화가 불가능하면 그 이유와 남는 사용자 리스크를 작업 마감 보고에 남긴다.

의도별 기준:

- `behavior`: 사용자 행동/화면 정책 보호 -> MSW/integration 허용
- `contract`: DTO/adapter/mapper 정합성 보호 -> typed fixture/typecheck 우선
- `connectivity`: 실제 서버 연결 안정성 확인 -> live smoke 필요

### 낮은 가치 테스트 예시

아래처럼 SDK wrapper가 서버 message를 그대로 반환하는지만 확인하는 테스트는 작성하지 않는다.

```txt
cancelPacking() 호출 -> response.message 반환
cancelPacking() 실패 -> 서버 message throw
```

이런 테스트는 transport 연결과 라이브러리 동작만 확인할 뿐, 비즈니스 정책 문서로 읽히기 어렵다.

### 파일명 원칙

테스트 파일명은 검증하려는 정책을 드러내야 한다.

권장:

```txt
factories.test.ts
mappers.test.ts
supplierCode.test.ts
domainFailurePolicy.test.ts
domainActionFlow.test.tsx
```

비권장:

```txt
api.test.ts
client.test.ts
reactQuery.test.ts
```

### 선호 테스트 예시

아래처럼 유스케이스 정책이 드러나는 테스트를 우선한다.

```txt
비동기 command 성공 시 화면은 필요한 query를 refetch하고 성공 메시지를 보여준다.
비동기 command 실패 시 화면은 에러 메시지를 보여주고 pending 상태를 해제한다.
지원되지 않는 문자열은 도메인 value object로 생성될 수 없다.
```

## 15. Definition of Done

한 도메인이 표준을 따른다고 보기 위한 최소 조건은 아래다.

1. UI direct `fetch` 없음
2. UI에서 generated 타입 직접 사용 없음
3. generated SDK 호출은 domain/api 내부에만 존재
4. query key factory 존재
5. mutationOptions factory 존재
6. domain type이 UI 진입점 역할을 함
7. 테스트 또는 명시적 테스트 공백 기록 존재

## 16. 예외 규칙

표준 예외는 허용할 수 있지만, 아래 조건을 만족해야 한다.

1. 왜 예외인지 설명할 수 있어야 한다.
2. 언제 제거할지 추적 가능해야 한다.
3. 레거시 호환 때문에 남겨두는 경우라도 신규 코드에 복제되면 안 된다.
4. 기존 코드의 불완전한 상태는 즉시 예외 근거가 될 수 있지만, 신규 코드의 기준을 낮추는 근거가 되어서는 안 된다.
5. 기존 코드는 표준 평가 대상이며, 단지 정리 순서가 다를 뿐 표준 밖으로 취급하지 않는다.

## 17. 참고 방식

구체적인 기준 도메인 예시와 현재 프로젝트 내 정렬 대상은 표준 문서가 아니라
reference-domain 문서와 계획 문서에서 관리한다.
