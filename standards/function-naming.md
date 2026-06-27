# 함수 네이밍 표준

## 원칙

함수명은 호출자가 알아야 하는 관심사만 드러낸다.

Generated SDK 이름은 codegen 산출물의 계약 이름일 뿐, 프론트엔드 레이어의
네이밍 source of truth가 아니다. Generated operation name이 업무 의도와
충돌하거나 `ForPackingV3`, `ForAnyStatus`, controller/service 이름처럼 내부 구현
세부사항을 드러내면, generated 경계 밖에서는 이 문서의 레이어별 네이밍 규칙을
우선한다.

React Query custom hook은 의도적인 예외다. 컴포넌트가 서버 상태 사용 방식을
예측할 수 있어야 하므로 `Query` 또는 `Mutation` 접미사를 사용한다. Hook
이름에서 `Query`와 `Mutation`은 숨겨야 할 구현 세부사항이 아니라 공개
계약이다.

## 선언 방식

컴포넌트와 hook은 `function` 선언을 기본으로 한다.

```ts
export function SearchField(props: SearchFieldProps) {
  // ...
}

export function useKeywordSearch() {
  // ...
}
```

변수, 상수, query/mutation factory, 순수 helper, mapper, event handler는
`const` 선언을 기본으로 한다.

```ts
const DEFAULT_PAGE_SIZE = 50;

const mapSupplierDtoToItem = (dto: SupplierDto): SupplierItem => ({
  id: dto.id,
  code: dto.supplier_code,
});

const handleSearchSubmit = () => {
  // ...
};
```

이 기준은 코드 탐색 시 “React 실행 단위”와 “값/헬퍼/조합 로직”을 빠르게
구분하기 위한 것이다. 컴포넌트와 hook은 hoisting과 stack trace에서 이름이
드러나는 `function` 선언을 사용하고, 값처럼 전달되거나 조합되는 로직은 `const`로
고정한다.

## 레이어별 규칙

### Generated SDK

생성 코드는 생성된 이름을 유지한다. 생성 함수명을 수동으로 바꾸지 않는다.
생성 함수명은 code generation을 통해서만 바뀌어야 한다.
단, 이 이름은 `src/api/generated/**`와 이를 직접 호출하는 좁은 adapter 내부에서만
그대로 둔다. `api.ts` export, `queries.ts`/`mutations.ts`, custom hook, UI 변수명,
event handler에는 generated 이름을 그대로 복사하지 않는다.

허용 예:

- `finishPacking`
- `updateReceiptForPackingV3`
- `batchScan`

### API Adapter Layer

예: `src/domain/*/api.ts`

API adapter 함수는 API 호출 레이어이므로 HTTP/transport 접두사를 허용한다.
Adapter는 API 호출 경계를 명확히 하기 위해 transport operation을 어느 정도
반영할 수 있다.

허용 예:

- `getPackingOrders`
- `postFinishPacking`
- `postPackingScan`
- `patchReceiptQuantity`
- `fetchPackingOrders`

피해야 할 예:

- `updateReceiptForPackingV3`: generated SDK 이름을 노출한다.
- `updateInvoiceNumbersForAnyStatus`: generated operation 조건을 caller-facing
  이름으로 노출한다. caller가 알아야 할 업무 의도는 `updateInvoiceNumbers`다.
- `batchScanAuditBarcodes`: batching이 호출자 관심사가 아닌데도 노출한다.
- `callWmsApi`: 업무 operation이 아니라 transport plumbing을 노출한다.

### Domain Query/Mutation Factory 또는 Service Layer

예: `queries.ts`, `mutations.ts`, use-case/service module

HTTP method를 숨기고 업무 동사를 사용한다. Query/mutation factory는 `api.ts`에서
사용한 transport verb가 아니라, 호출자가 선택하는 업무 operation을 드러낸다.

허용 예:

- `packingMutations.finish()`
- `packingMutations.scan()`
- `packingMutations.checkDeliveryRestriction()`

피해야 할 예:

- `packingMutations.postFinish()`
- `packingMutations.patchReceipt()`
- `packingMutations.batchScan()`
- `orderInvoiceMutations.updateInvoiceNumbersForAnyStatus()`: generated operation
  이름을 domain factory에 노출한다.

### React Query Custom Hook Layer

React Query hook은 업무 의도에 `Query` 또는 `Mutation` 접미사를 붙인다.
`Post`, `Patch`, `Fetch`, `Batch` 같은 transport/infrastructure 용어는 숨긴다.

허용 예:

- `usePackingOrdersQuery`
- `useFinishPackingMutation`
- `usePackingScanMutation`
- `useReceiptAuditScanSubmissionMutation`
- `useReceiptAuditDatasetQuery`

피해야 할 예:

- `usePostFinishPacking`
- `useFetchPackingOrders`
- `usePatchReceiptQuantity`
- `useBatchScanAuditMutation`

### UI에서 직접 쓰는 React Query Mutation 변수

컴포넌트가 `useMutation()`을 직접 호출하면 mutation result object를 업무 의도 +
`Mutation` 접미사로 이름 짓는다. 이렇게 하면 loading, error, success, reset,
mutate 접근이 하나의 변수에서 예측 가능해진다.

허용 예:

- `const cancelOrderMutation = useMutation(orderCancelMutations.cancelOrder())`
- `const finishPackingMutation = useMutation(packingMutations.finish())`
- `cancelOrderMutation.mutate(payload)`
- `cancelOrderMutation.isPending`

피해야 할 예:

- `const { mutate: cancelOrder } = useMutation(orderCancelMutations.cancelOrder())`
- `const { mutate: mutateCancelOrder } = useMutation(orderCancelMutations.cancelOrder())`

렌더링 가독성을 높이기 위한 status field의 작은 local destructuring은 허용한다.
다만 새 코드에서 `mutate`는 mutation object에 남겨 둔다.

### UI / State Hook Layer

서버 상태 hook이 아닌 hook은 `Query` 또는 `Mutation` 접미사를 사용하지 않는다.
관리하는 local UI state 또는 workflow를 기준으로 이름 짓는다.

허용 예:

- `useBarcodeInput`
- `useReceiptAuditDataset`
- `useRowQtyEdit`
- `useCancelMode`

### UI Event Handlers와 Callback Props

UI interaction wiring 이름은 **구현체와 인터페이스를 구분**한다.

- 컴포넌트/훅 내부에서 정의하고 직접 실행 흐름을 조합하는 handler 함수는
  `handleX`를 사용한다.
- 컴포넌트가 외부에서 받는 callback prop은 React public API 관례에 맞춰 `onX`를
  사용한다.
- `handleX`는 구현체 이름이므로 props interface, JSX prop, props object bag에
  노출하지 않는다.
- local editing workflow는 `startEditing`, `stopEditing`처럼 상태 전이를 드러내는
  동사를 사용할 수 있다.

허용 예:

- `handleQtyInputFocus`
- `handleSearchSubmit`
- `onSubmit`
- `onValueChange`
- `startEditing`
- `stopEditing`

피해야 할 예:

- `interface Props { handleSubmit: () => void }`
- `<SearchSection handleSubmit={handleSearchSubmit} />`
- `search={{ handleSubmit: handleSearchSubmit }}`

### Mapper와 Model Utility

변환 의도가 드러나는 이름을 사용한다.

허용 예:

- `mapReceiptToRow`
- `buildDeliveryRestrictionCheckResult`
- `toReceiptItem`
- `fromReceiptResponse`

## 요약

- `api.ts`: `get`, `post`, `put`, `patch`, `fetch`, `delete`를 허용한다.
- `queries.ts` / `mutations.ts`: HTTP/transport 단어가 아니라 업무 의도를
  사용한다.
- React Query custom hook: 업무 의도 + `Query` 또는 `Mutation`을 사용한다.
- UI/local state hook: 업무 또는 상태 의도만 사용한다. `Query`나 `Mutation`을
  붙이지 않는다.
- 모든 레이어는 generated name과 `batch`, `callApi`, `ForPackingV3` 같은 내부
  구현 세부사항이 호출자 관심사가 아니면 숨긴다.
- Generated SDK 이름과 레이어별 네이밍 규칙이 충돌하면 generated 경계 밖에서는
  레이어별 네이밍 규칙을 우선한다.
- 얇은 매핑/alias wrapper는 만들지 않는다. 자세한 기준은 `## 얇은 매핑 함수 지양` 섹션 참조.

## 얇은 매핑 함수 지양

함수가 의미 있는 변환·검증·추상화·시그니처 변환을 하지 않고 다른 함수에 인자를 그대로 전달하기만 하는 wrapper는 만들지 않는다.

### 금지 패턴

- 동일 함수에 대한 단순 alias를 객체 키만 달리 해서 노출:

```ts
return {
  setMode,
  handleModeChange: setMode, // 금지
};
```

- dispatcher에 키 이름만 채우고 다른 일은 하지 않는 setter wrapper:

```ts
const setMode = (next: StockInquiryMode) => updateFilters({ mode: next }); // 금지
```

### 허용되는 wrapper

다음 중 한 가지 이상을 충족하면 wrapper를 만들 수 있다.

- 호출자 타입을 의미 있게 좁힌다. 예: 광범위 dispatcher를 좁은 도메인 타입 setter로 감싸는 경우.
- 호출자/콜백 시그니처 어댑터. 예: prop callback `(next: string) => void`와 내부 dispatcher `(updates: Partial<X>) => void` 사이 시그니처 변환.
- 검증·변환·정규화·부작용을 함께 묶은 경우.
- 여러 호출을 하나의 의도로 묶는 경우.

### 권장 패턴

의미 있는 dispatcher 하나를 외부에 노출하고, 호출자가 의도된 키로 직접 호출한다. UI 이벤트 핸들러는 호출하는 컴포넌트 내 로컬 `handleX` 래퍼로 두고 prop은 `onX`로 받는다.

```ts
// 훅 쪽
return { mode, updateFilters };

// 호출자 쪽
const { mode, updateFilters } = useStockInquiryFilters();
updateFilters({ mode: 'sales' });
updateFilters({ sortField: 'totalStockQuantity', sortDirection: 'desc' });
```

이 규칙은 React `useState`의 `[value, setValue]` 관례를 부정하지 않는다. `useState`가 반환하는 `setX`는 React 자체 라이브러리 컨벤션이고, 이 규칙은 새로 추가하는 wrapper 함수에만 적용한다.

## Enforcement

ESLint rule `sirloin/layered-function-naming`은 이 표준 중 신호가 명확한 일부 규칙을
강제한다.

- Generated file은 무시한다.
- `src/domain/*/api.ts`는 `get`, `post`, `put`, `patch`, `fetch`, `delete`
  접두사를 가진 API adapter 이름을 export할 수 있다.
- Non-API domain export는 `post`, `put`, `patch`, `fetch` 같은 transport
  접두사로 시작하지 않아야 한다.
- Hook 이름은 `usePost`, `usePut`, `usePatch`, `useFetch`, `useBatch`로
  시작하지 않아야 한다.
- API/domain exported function은 `batch`, `callApi`, `edit`,
  `ForPackingV3` 같은 내부 구현명이나 generated-version naming을 노출하지
  않아야 한다.
- UI callback prop contract는 `on*`로 이름 지어야 한다. `handle*`는 로컬 구현체
  함수명으로만 사용하고 props interface, JSX prop, object prop bag에 노출하지
  않아야 한다.

이 rule은 의도적으로 보수적이다. 모든 유효한 업무 동사를 추론하려 하지 않고,
레이어 관심사 누수를 포착하는 데 집중한다.
