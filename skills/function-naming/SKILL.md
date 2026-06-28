---
name: layered-function-naming
description: 함수 네이밍, 접두사 컨벤션, API/domain/hook 레이어별 이름 검토 또는 리팩터링 시 사용. API adapter, domain factory/service, React Query hook, UI/state hook 레이어별로 호출자 관심사만 드러내는 함수명을 적용한다.
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# Layered Function Naming

범용 프로젝트에 레이어별 함수 네이밍 컨벤션을 적용한다.

기준 문서: `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`

## Trigger

이 스킬은 다음 요청에서 사용한다:

- 함수명, 접두사, naming convention, 네이밍 리뷰
- `get` / `fetch` / `post` / `create` / `update` / `patch` / `add` / `edit` 선택
- API wrapper, domain service, React Query hook, UI/state hook 이름 정리
- 레이어별 네이밍 lint 또는 review 위반 수정

## Core Rule

함수명은 호출자가 알아야 하는 관심사만 드러낸다.

단, React Query custom hook은 팀 표준상 `Query` / `Mutation` 접미사를 붙인다.
여기서 `Query` / `Mutation`은 숨길 구현이 아니라, 컴포넌트가 서버 상태 훅의
사용 방식을 예측하기 위한 인터페이스 신호다.

- Generated SDK: 생성된 이름 유지
- API adapter: HTTP/transport 접두사 허용
- Domain query/mutation factory 또는 service: HTTP method를 숨기고 업무 동사 사용
- React Query custom hook: 업무 의도 + `Query`/`Mutation`
- UI/state hook: 업무/상태 의도만 사용, `Query`/`Mutation` 접미사 없음
- UI handler: `handle`, `on`, local editing verb 허용
- Mapper/model utility: `map`, `build`, `to`, `from`

## Layer Rules

### Generated SDK

생성 코드이므로 이름을 바꾸지 않는다.

Allowed:

- `finishCheckout`
- `updateCustomerProfileV2`
- `batchSubmit`

### API Adapter Layer

예: `domain/*/api.ts`, `services/*/api.ts`, `lib/api/*`

API 호출 레이어이므로 HTTP/transport 접두사를 허용한다.

Allowed:

- `getProductList`
- `postCheckout`
- `postItemScan`
- `patchProfile`
- `fetchProductList`

Avoid:

- `updateCustomerProfileV2` - generated 이름 노출
- `batchSubmitItems` - batch가 호출자 관심사가 아니면 피함
- `callExternalApi` - transport plumbing 노출

### Domain Query/Mutation Factory 또는 Service Layer

예: `queries.ts`, `mutations.ts`, use-case/service module.

HTTP method는 숨기고 업무 동사를 사용한다.

Allowed:

- `checkoutMutations.finish()`
- `inventoryMutations.scan()`
- `customerMutations.updateProfile()`

Avoid:

- `checkoutMutations.postFinish()`
- `customerMutations.patchProfile()`
- `inventoryMutations.batchScan()`

### React Query Custom Hook Layer

React Query hook이면 업무 의도에 `Query` / `Mutation` 접미사를 붙인다.
단, `Post`, `Patch`, `Fetch`, `Batch` 같은 transport/infra 단어는 숨긴다.

Allowed:

- `useProductListQuery`
- `useFinishCheckoutMutation`
- `useItemScanMutation`
- `useProfileUpdateMutation`
- `useCustomerDatasetQuery`

Avoid:

- `usePostFinishCheckout`
- `useFetchProductList`
- `usePatchProfile`
- `useBatchScanItemsMutation`

### UI에서 직접 쓰는 `useMutation()` 변수

컴포넌트가 `useMutation()`을 직접 호출하면 결과 객체를 업무 의도 + `Mutation`
접미사로 유지한다. 이 변수는 실행 함수가 아니라 `mutate`, `isPending`,
`isError`, `error`, `isSuccess`, `reset`을 함께 가진 서버 command 상태 객체다.

Allowed:

- `const cancelItemMutation = useMutation(itemMutations.cancel())`
- `cancelItemMutation.mutate(payload)`
- `cancelItemMutation.isPending`

Avoid:

- `const { mutate: cancelItem } = useMutation(itemMutations.cancel())`
- `const { mutate: mutateCancelItem } = useMutation(itemMutations.cancel())`

상태 필드는 렌더링 가독성을 위해 가까운 곳에서 구조분해할 수 있지만, 새 코드에서
`mutate`는 alias로 빼지 않는다.

### UI / State Hook Layer

서버 상태 훅이 아니면 `Query` / `Mutation`을 붙이지 않는다.

Allowed:

- `useSearchInput`
- `useCustomerDataset`
- `useRowEdit`
- `useSelectionMode`

### UI Event Handlers와 Callback Props

UI interaction wiring은 구현체와 인터페이스를 구분한다.

- 내부 구현체 handler 함수: `handleX`
- 외부에서 받는 callback prop/public API: `onX`
- local editing workflow: `startEditing`, `stopEditing` 같은 상태 전이 동사

Allowed:

- `handleQuantityInputFocus`
- `handleSearchSubmit`
- `onSubmit`
- `onValueChange`
- `startEditing`
- `stopEditing`

Avoid:

- `interface Props { handleSubmit: () => void }`
- `<SearchSection handleSubmit={handleSearchSubmit} />`
- `search={{ handleSubmit: handleSearchSubmit }}`

## Workflow

1. 함수가 어느 레이어에서 export/import 되는지 확인한다.
2. `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`의 해당 레이어 규칙을 적용한다.
3. 이름 변경 시 import/export와 테스트를 함께 갱신한다.
4. 프로젝트에 네이밍 lint가 있으면 대상 파일에 실행한다. 없으면 타입체크/테스트로 참조 갱신을 검증한다.

최소 검증 예:

```bash
./node_modules/.bin/eslint path/to/file.ts
```

## Lint Rule Behavior

전용 lint rule이 있는 프로젝트라면 보수적으로 다음을 강제하는지 확인한다:

- generated 파일 무시
- `api.ts` 또는 adapter boundary에서는 `get`, `post`, `put`, `patch`, `fetch`, `delete` 접두사 허용
- non-API domain export에서 `post`, `put`, `patch`, `fetch` 접두사 금지
- hook 이름에서 `usePost`, `usePut`, `usePatch`, `useFetch`, `useBatch` 금지
- API/domain exported function에서 `batch`, `callApi`, version suffix 같은 내부 구현명 금지
- UI callback prop contract에서 `handle*` 이름 금지. 공개 props는 `on*`, 로컬 구현체는 `handle*`를 사용

규칙이 걸리면 disable comment보다 레이어 경계의 함수명을 고친다. 예외가 필요하면 호환성 이유를 코드 근처에 짧게 남긴다.
