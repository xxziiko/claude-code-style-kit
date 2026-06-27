# URL state 표준

> Sirloin OMS Frontend 의 URL state 관리 표준. 신규 페이지/Drawer 는 이 표준을 따른다.

- Status: Baseline v1
- Scope: Repo-wide 신규 코드

## 1. 원칙 (5가지, 위반 금지)

1. URL 은 사용자 가시 상태의 single source of truth. 필터, 탭, 페이지네이션, drawer 개폐, drawer 식별자, drawer 내부 검색어 등 새로고침/뒤로가기에서 보존되어야 하는 모든 것은 URL 에 직렬화한다.
2. `useState` 는 typing 중 draft, 순수 UI 임시 상태에만 허용. 그 외는 URL.
3. URL ↔ 도메인 변환은 slice 단위로, 합성층에서만 일어난다. 도메인 코드에서 `params.set` / `params.delete` 호출 금지.
4. slice 간 정합 invariant 는 `normalize` 한 함수에서만 강제한다. reducer 본문에 if 분기 금지.
5. URL 에는 식별자만 직렬화한다. 표시 데이터(이름, 수량, 라벨)는 query 결과에서 lookup 한다.

## 2. 레이어

```
L1  원자 어댑터       src/shared/utils/urlParams/paramParsers.ts                (기존, 변경 없음)
L2  합성층            src/shared/hooks/url/
                        useUrlState.ts                                          (앱 전체 단일 hook)
                        useUrlSelection.ts                                      (URL keys → 풍부 객체)
                        slices/                                                 (슬라이스 빌더 카탈로그)
L3  도메인            src/features/<area>/<feature>/
                        <feature>UrlStateSchema.ts                              (UrlStateSchema 객체 1개 + drawer URL keys)
                        utils/resolve<Selection>.ts                             (순수 함수)
```

L3 는 L2 만 호출한다. L3 에서 `useSearchParams` / `useTransition` 직접 호출 금지. **L3 에 wrapper hook (`use<Feature>UrlState`, `use<Feature>UrlParams`) 을 만들지 않는다.**

## 3. 도메인이 작성하는 것 — typed schema 객체 1개

페이지/기능마다 작성하는 것은 plain 객체 하나. 도메인별 hook 함수는 만들지 않는다.

```ts
export const outboundOrderListUrlStateSchema: UrlStateSchema<
  OutboundOrderListUrlState,
  typeof reducers
> = {
  slices: { ... },
  reducers,
  normalize: ...,   // 선택
  legacyKeys: [...], // 선택
};
```

페이지 컴포넌트는 그 객체를 `useUrlState` 에 넘긴다.

```tsx
const { state, actions } = useUrlState(outboundOrderListUrlStateSchema);
```

진짜 React hook 은 앱 전체에서 `useUrlState`, `useUrlSelection` 두 개뿐.

## 4. L2 API 시그니처 (확정)

### 4.1 `UrlStateSchema`

```ts
export interface UrlStateSchema<
  TState extends Record<string, unknown>,
  TReducers extends Record<
    string,
    (state: TState, ...payload: never[]) => TState
  >,
> {
  slices: { [K in keyof TState]: UrlSlice<TState[K]> };
  reducers: TReducers;
  normalize?: (state: TState) => TState;
  legacyKeys?: readonly string[];
}
```

- `slices`: 각 슬라이스의 owned URL keys, parse/serialize 를 캐프슐화.
- `reducers`: 순수 함수. `(state, ...payload) => nextState`. React 와 무관.
- `normalize`: slice 간 정합 invariant. parse 직후, serialize 직전에 호출.
- `legacyKeys`: 페이지가 청소해야 할 옫 잔존 키. 쓰기 시점에 delete.

### 4.2 `useUrlState`

```ts
export function useUrlState<TState, TReducers>(
  schema: UrlStateSchema<TState, TReducers>,
): {
  state: TState;
  actions: {
    [K in keyof TReducers]: (...payload: ReducerPayloads<TReducers[K]>) => void;
  };
  isPending: boolean;
};
```

내부 동작 (L2 한 곳에만 존재):

- `useSearchParams` + `useTransition` 한 번 호출.
- parse: 각 slice 의 `parse` → 합쳐 `TState` 구성 → `normalize` 적용.
- 각 `actions[K]` 호출 시 `startTransition` 내에서 `setSearchParams((prev) => …)`:
  1. prev 복사
  2. 모든 slice 의 `ownedKeys` + `legacyKeys` 를 delete
  3. `reducer(state, ...payload)` 적용
  4. `normalize` 호출
  5. 각 slice 의 `serialize` 호출

### 4.3 슬라이스 빌더

`src/shared/hooks/url/slices/` 에 다음만 둔다. 도메인 코드는 이 5종으로만 slice 를 만든다.

| 빌더                                    | 용도                                                                                                                                        |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| `singleKeySlice(key, adapter)`          | URL 키 1개 ↔ 값 1개.                                                                                                                        |
| `objectSlice(map)`                      | 여러 키를 한 객체로. URL 키와 객체 property 이름이 같을 때 사용.                                                                            |
| `prefixedObjectSlice(prefix, map)`      | 객체 property 는 prefix 없이 유지하고 URL 키에만 prefix 를 붙인다. Drawer detail keys 처럼 `detailStockItemId` ↔ `stockItemId` 매핑에 사용. |
| `paginationSlice(config)`               | 기존 `paginationUtils` 래핑. 1-based / 0-based 지원.                                                                                        |
| `discriminatedSlice(kindKey, variants)` | open/close + 분기되는 drawer 용. `null                                                                                                      | { kind, … }`. |

새 빌더가 필요하면 L2 에 추가한다. L3 도메인 코드에 inline `parse` / `serialize` 함수 작성 금지.

### 4.4 `useUrlSelection`

```ts
export function useUrlSelection<TKeys, TResolved>(args: {
  keys: TKeys | null;
  resolve: (keys: TKeys) => TResolved | null;
}): TResolved | null;
```

URL 식별자(keys) 와 query 데이터에서 풍부 객체를 도출한다. `resolve` 가 null 이면 drawer 가 자연스럽게 닫힌다.

## 5. 파일 레이아웃

```
src/features/<area>/<feature>/
  <feature>UrlStateSchema.ts                  UrlStateSchema 객체 1개
  ui/
    <Page>View.tsx                            presentational
    <Drawer>.tsx                              presentational
  utils/
    resolve<Selection>.ts                     URL keys + query → 풍부 객체
  types.ts
  index.ts
```

`<feature>UrlStateSchema.ts` 는 hook 이 아니므로 `hooks/` 에 넣지 않는다.

## 6. drawer URL 계약은 schema 에 둔다

```ts
// outboundOrderListUrlStateSchema.ts
interface OutboundOrderDetailKeys {
  orderId: string;
}

const outboundOrderDetailDrawerSlice = discriminatedSlice('detail', {
  order: prefixedObjectSlice('detail', { orderId: stringParam() }),
});
```

규칙:

- selection 타입에는 식별자만 둔다. 상품명, 수량, dayOffset 같은 표시 데이터 금지.
- drawer 컴포넌트는 URL 을 모른다. props 만 받는다.
- 페이지 컴포넌트가 hook → props 매핑을 책임진다.

## 7. 네이밍 규칙

| 종류                     | 형식                         | 예시                                             |
| ------------------------ | ---------------------------- | ------------------------------------------------ |
| Schema 객체              | `<feature>UrlStateSchema`    | `outboundOrderListUrlStateSchema`                |
| URL 상태 타입            | `<Feature>UrlState`          | `OutboundOrderListUrlState`                      |
| Drawer slice 변수        | `<drawerCamel>Slice`         | `outboundOrderDetailDrawerSlice`                 |
| Drawer URL keys 타입     | `<Feature><Drawer>Keys`      | `OutboundOrderDetailKeys`                        |
| Drawer resolve 결과 타입 | `<Feature><Drawer>Selection` | `OutboundOrderDetailSelection`                   |
| Resolve 함수             | `resolve<Selection>`         | `resolveOutboundOrderSelection`                  |
| Action 이름              | 도메인 동사 + 명사           | `selectStatus`, `openOrderDetail`, `closeDrawer` |

금지 네이밍:

- `use<Feature>UrlState`, `use<Feature>UrlParams` — wrapper hook 자체 금지.
- `define...`, `create...`, `make...` — 헬퍼/축약 네이밍 금지.
- `handle...`, `set<Slice>`, `update<Slice>` — action 은 사용자 의도 동사로 짓는다.

## 8. 신규 페이지 / drawer 체크리스트

### A. 새 페이지 (drawer 없음)

1. `<Feature>UrlState` 타입 정의.
2. reducers 맵 정의 (`(state, ...payload) => state`).
3. `<feature>UrlStateSchema.ts` 작성: slices + reducers + (필요 시) normalize.
4. 페이지 컴포넌트에서 `useUrlState(schema)` 호출.
5. View 컴포넌트는 props 만 받는 presentational.

### B. 새 drawer

1. `<feature>UrlStateSchema.ts` 안에 `discriminatedSlice` + drawer keys 타입을 둔다.
2. Schema 의 `slices.detail` 에 drawer slice 추가.
3. Schema 의 `reducers` 에 `openXxx`, `closeDrawer` 추가.
4. `resolve<Selection>` 순수 함수 작성.
5. 페이지에서 `useUrlSelection({ keys: state.detail, resolve })` 로 풍부 객체 얻기.
6. drawer 컴포넌트는 selection 을 props 로 받는다. URL hook 호출 금지.

### C. 새 action

1. Schema 의 `reducers` 에 reducer 한 줄 추가.
2. invariant 가 영향 받으면 `normalize` 갱신. reducer 안에 if 분기 금지.

### D. 새 slice 빌더가 필요할 때

1. 멈춘다. 5종 빌더로 정말 표현 불가한지 재검토.
2. 진짜 안 되면 L2 (`slices/`) 에 빌더 추가. L3 에 inline 작성 금지.

## 9. 금지 사항 (PR reject 대상)

- 도메인 코드에서 `useSearchParams`, `useTransition` 호출.
- 도메인별 wrapper hook (`use<Feature>UrlState` / `use<Feature>UrlParams`) 작성.
- L3 에 `parseX` / `serializeX` 함수 작성.
- selection 타입에 표시 데이터 (이름, 수량, 라벨) 포함.
- reducer 안의 invariant `if` 분기. → `normalize` 로.
- URL 동기화용 `useEffect(() => setSearchParams(...), [state])`. URL 이 SoT 면 sync 가 필요 없다.
- `useState` 로 drawer 개폐 추적. 무조건 URL slice.
- 헬퍼 이름 축약 (`define...`, `create...`, `make...`).
- **Immer 도입 금지**. reducer 는 plain spread 로만 작성. state 깊이가 심해서 spread 노이즈가 문제 되는 페이지가 실제로 등장하면 별도 안건으로 Immer 도입을 재검토한다.
- “기존이 그래서” 신규 코드 표준 위반.

## 10. 워크드 예제 — 출고 지시 목록 + 상세 drawer

### 10.1 drawer keys in `<feature>UrlStateSchema.ts`

```ts
import { discriminatedSlice, objectSlice } from '@shared/hooks/url/slices';
import { stringParam } from '@shared/utils/urlParams/paramParsers';

interface OutboundOrderDetailKeys {
  orderId: string;
}

const outboundOrderDetailDrawerSlice = discriminatedSlice('detail', {
  order: objectSlice({
    orderId: stringParam(),
  }),
});
```

### 10.2 `outboundOrderListUrlStateSchema.ts`

```ts
import type { UrlStateSchema } from '@shared/hooks/url/useUrlState';
import { objectSlice, paginationSlice } from '@shared/hooks/url/slices';
import { enumParam } from '@shared/utils/urlParams/paramParsers';

const ORDER_STATUS_VALUES = ['all', 'pending', 'processing', 'done'] as const;
type OutboundOrderStatus = (typeof ORDER_STATUS_VALUES)[number];

export interface OutboundOrderListUrlState {
  filter: { status: OutboundOrderStatus; categoryId: string | null };
  pagination: { pageIndex: number; pageSize: number };
  detail: { kind: 'order'; orderId: string } | null;
}

const reducers = {
  selectStatus: (
    state: OutboundOrderListUrlState,
    status: OutboundOrderStatus,
  ): OutboundOrderListUrlState => ({
    ...state,
    filter: { ...state.filter, status },
    pagination: { ...state.pagination, pageIndex: 0 },
  }),

  selectCategory: (
    state: OutboundOrderListUrlState,
    categoryId: string | null,
  ): OutboundOrderListUrlState => ({
    ...state,
    filter: { ...state.filter, categoryId },
    pagination: { ...state.pagination, pageIndex: 0 },
  }),

  openOrderDetail: (
    state: OutboundOrderListUrlState,
    orderId: string,
  ): OutboundOrderListUrlState => ({
    ...state,
    detail: { kind: 'order', orderId },
  }),

  closeDrawer: (
    state: OutboundOrderListUrlState,
  ): OutboundOrderListUrlState => ({
    ...state,
    detail: null,
  }),
};

export const outboundOrderListUrlStateSchema: UrlStateSchema<
  OutboundOrderListUrlState,
  typeof reducers
> = {
  slices: {
    filter: objectSlice({
      status: enumParam(ORDER_STATUS_VALUES, 'all'),
      categoryId: nullableStringParam(),
    }),
    pagination: paginationSlice({
      keys: { pageIndex: 'page', pageSize: 'limit' },
      base: 1,
      defaultPageSize: 30,
    }),
    detail: outboundOrderDetailDrawerSlice,
  },
  reducers,
};
```

### 10.3 `utils/resolveOutboundOrderSelection.ts`

```ts
import type { OutboundOrder, OutboundOrderListView } from '../types';
import type { OutboundOrderDetailKeys } from '../types';

export function resolveOutboundOrderSelection(
  keys: { kind: 'order' } & OutboundOrderDetailKeys,
  list: OutboundOrderListView,
): OutboundOrder | null {
  return list.items.find((item) => item.id === keys.orderId) ?? null;
}
```

### 10.4 `pages/OutboundOrderListPage.tsx`

```tsx
import { useUrlSelection } from '@shared/hooks/url/useUrlSelection';
import { useUrlState } from '@shared/hooks/url/useUrlState';

import { outboundOrderListUrlStateSchema } from '@features/outbound/orderList/outboundOrderListUrlStateSchema';
import { resolveOutboundOrderSelection } from '@features/outbound/orderList/utils/resolveOutboundOrderSelection';

export default function OutboundOrderListPage() {
  const { state, actions } = useUrlState(outboundOrderListUrlStateSchema);

  const listQuery = useOutboundOrderListQuery({
    ...state.filter,
    ...state.pagination,
  });

  const selection = useUrlSelection({
    keys: state.detail,
    resolve: (keys) => resolveOutboundOrderSelection(keys, listQuery.data),
  });

  return (
    <>
      <OutboundOrderListView
        items={listQuery.data.items}
        filter={state.filter}
        onStatusChange={actions.selectStatus}
        onCategoryChange={actions.selectCategory}
        onOpenDetail={actions.openOrderDetail}
      />
      <OutboundOrderDetailDrawer
        selection={selection}
        onClose={actions.closeDrawer}
      />
    </>
  );
}
```

## 11. 도메인이 절대 작성하지 않는 것

다음은 모두 `useUrlState` 가 한 번에 처리한다. 도메인 코드에 등장하면 패턴 위반:

- `useSearchParams`, `useTransition` 호출
- `setSearchParams((prev) => …)` 클로저
- `params.set`, `params.delete` 직접 호출
- slice ownedKeys / legacyKeys 청소 루프
- `parseX`, `serializeX` 함수 정의
- `(payload) => update(state => reducer(state, payload))` 래핑
- drawer 개폐 직렬화
- `normalize` 호출 타이밍 관리
- action 별 `useCallback` 래핑

## 12. 도메인이 반드시 작성하는 것 (irreducible)

도메인 본질이라 어떤 추상화로도 사라지지 않는다.

- URL 상태 타입
- slice 선언 (URL 키 + 어댑터)
- reducer 본문 `(state, ...payload) => nextState`
- (있다면) `normalize` 함수
- drawer URL slice 와 selection 식별자 타입
- `resolve<Selection>` 순수 함수

## 13. 마이그레이션 정책

- 신규 코드는 무조건 본 표준.
- 기존 hook (`useStockOverviewUrlParams`, `useDateBucketStockUrlParams`, `useDateBucketDetailSelection`, `useDailyStockUrlParams`, `useReceiptHistoryQueryParams`, `useShipmentRestrictedUrlParams`, `useStockStatusQueryParams` 등) 은 별도 refactor queue. 해당 feature 를 만질 때 같이 이전. 빅뱅 금지.
