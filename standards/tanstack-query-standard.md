# TanStack Query 적용 표준

> TanStack Query를 이 프로젝트에서 어떤 기준으로 적용할지 정의하는 문서

- Status: Baseline v1 (Agreed)
- Date: 2026-04-24
- Scope: Repo-wide

## 1. 목적

이 문서는 서버 상태를 TanStack Query로 다룰 때의 적용 기준을 통일한다.

핵심 목표:

1. query와 mutation의 기준을 일관되게 만든다.
2. 조회는 가능한 한 `useSuspenseQuery` 중심으로 통일한다.
3. domain과 호출부의 책임을 분리한다.
4. 하네스와 스캐폴더가 같은 기준을 따르도록 한다.

## 2. 기본 원칙

### Query / Mutation 분류

- `GET` 성격의 읽기 작업 -> query
- `POST`, `PUT`, `PATCH`, `DELETE` 성격의 쓰기/명령 작업 -> mutation

여기서 핵심은 HTTP 메서드만이 아니라 유스케이스의 성격이다.
다만 현재 프로젝트에서는 위 기준을 기본값으로 사용한다.

### 적용 우선순위

1. 조회는 기본적으로 `useSuspenseQuery`
2. 명령은 기본적으로 `useMutation`
3. query/mutation 정의는 domain
4. query invalidation / alert / navigation 같은 후처리는 호출부 또는 UI orchestration hook

## 3. Query 표준

### 기본 원칙

- 조회는 `queryOptions()` factory로 정의한다.
- 호출부는 기본적으로 `useSuspenseQuery()`를 사용한다.
- query key는 factory로 관리한다.
- query key의 root는 endpoint나 API 번호가 아니라 domain/aggregate 기준으로 둔다.

예:

```ts
export const domainQueries = {
  list: (params: DomainListParams) =>
    queryOptions({
      queryKey: domainKeys.list(params),
      queryFn: () => fetchDomainList(params),
    }),
};
```

호출부:

```ts
const { data } = useSuspenseQuery(domainQueries.list(params));
```

### Query Key 계층

query key는 서버 API의 물리적 endpoint 목록을 그대로 반영하지 않는다.
UI와 mutation 후처리가 다루는 캐시 경계를 기준으로 domain/aggregate root 아래에
하위 리소스를 배치한다.

권장:

```ts
export const receiptAuditKeys = {
  all: ['receiptAudit'] as const,
  details: () => [...receiptAuditKeys.all, 'detail'] as const,
  detail: (receiptAuditId: string) =>
    [...receiptAuditKeys.details(), receiptAuditId] as const,
  unregisteredBarcodes: (receiptAuditId: string) =>
    [...receiptAuditKeys.detail(receiptAuditId), 'unregisteredBarcodes'] as const,
  unregisteredBarcodeList: (params: UnregisteredBarcodeParams) =>
    [...receiptAuditKeys.unregisteredBarcodes(params.receiptAuditId), params] as const,
  adjustments: (receiptAuditId: string) =>
    [...receiptAuditKeys.detail(receiptAuditId), 'adjustments'] as const,
  adjustmentList: (params: ReceiptAuditAdjustmentParams) =>
    [...receiptAuditKeys.adjustments(params.receiptAuditId), params] as const,
};
```

이렇게 두면 mutation 후처리에서 캐시 범위를 의도에 맞게 선택할 수 있다.

```ts
queryClient.invalidateQueries({ queryKey: receiptAuditKeys.all });
queryClient.invalidateQueries({
  queryKey: receiptAuditKeys.detail(receiptAuditId),
});
queryClient.invalidateQueries({
  queryKey: receiptAuditKeys.adjustments(receiptAuditId),
});
```

피한다:

```ts
export const receiptAuditDetailKeys = {
  all: ['receiptAuditDetail'] as const,
};

export const pendingReceiptKeys = {
  all: ['pendingReceipt'] as const,
};

export const receiptAuditAdjustmentKeys = {
  all: ['receiptAuditAdjustment'] as const,
};
```

위처럼 API별로 독립 root를 만들면 같은 receipt audit aggregate에 속한 캐시를
한 번에 invalidate하기 어렵고, mutation 후처리에서 문자열 key를 조합하거나
여러 root를 나열하게 된다.

별도 domain으로 분리해도 되는 경우는 캐시 수명, invalidation 범위, 화면 소유권이
실제로 독립적인 경우다. 단지 endpoint가 다르다는 이유만으로 root를 나누지 않는다.

### `useSuspenseQuery`를 기본으로 두는 이유

- loading/error 처리를 boundary와 함께 일관되게 가져갈 수 있다.
- 조회형 화면의 상태 흐름이 더 단순해진다.
- query-only 화면에서 로컬 loading 분기가 줄어든다.

### 페이지네이션 / 필터 기반 쿼리

URL 파라미터나 필터 변경에 따라 query key가 바뀌는 화면에서는 호출부에서 `placeholderData: keepPreviousData`를 적용한다.

- 필터/페이지 전환 시 이전 데이터를 유지해 UI 깜빡임을 방지한다.
- domain `queryOptions`에는 두지 않는다 — 화면 UX 관심사이기 때문이다.
- invalidate 후에는 placeholder 없이 즉시 새 데이터로 전환된다.

```ts
import { keepPreviousData, useQuery } from '@tanstack/react-query';

// rest-query-exception: useQuery - URL 파라미터 기반 필터 조합으로 로딩 상태를 직접 제어
const { data } = useQuery({
  ...domainQueries.list(toListParams(searchParams)),
  placeholderData: keepPreviousData,
});
```

## 4. `useQuery` 예외 기준

아래 경우에는 `useQuery`를 허용한다.

1. 조건부 조회가 필요할 때
2. polling / 주기적 refetch가 필요할 때
3. suspense boundary 적용이 부적절한 legacy 화면
4. 사용자가 조회 타이밍을 명시적으로 제어해야 할 때

즉, `useQuery`는 금지가 아니라 예외 경로다.
예외 경로를 선택하면 호출부 근처에 사유를 남겨야 한다.

```ts
// rest-query-exception: useQuery - polling 조회라 suspense boundary와 분리한다
const query = useQuery(domainQueries.pollingStatus(params));
```

변경된 UI 파일에서 TanStack `useQuery`를 사유 없이 쓰면 `standards:check`가 실패한다.

## 5. Mutation 표준

### 기본 원칙

- mutation은 `mutationOptions()` factory로 domain에서 정의한다.
- 호출부는 `useMutation(domainMutations.xxx())`를 직접 호출한다.
- domain mutation은 기본적으로 `mutationFn`만 소유한다.
- `mutationKey`는 mutation identity를 실제로 소비할 때만 선택적으로 둔다.

예:

```ts
export const domainMutations = {
  execute: () =>
    mutationOptions({
      mutationFn: (payload: DomainCommandPayload) =>
        executeDomainCommand(payload),
    }),
};
```

`mutationKey`가 필요한 경우:

- `useMutationState` / `isMutating`으로 특정 mutation 상태를 추적한다.
- `queryClient.setMutationDefaults` 같은 mutation defaults를 사용한다.
- 공통 pending indicator가 mutation identity에 의존한다.

단순 command 실행, 성공/실패 alert, invalidate에는 `mutationKey`가 필요하지 않다.

호출부:

```ts
const executeMutation = useMutation({
  ...domainMutations.execute(),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: domainKeys.lists() });
  },
});

executeMutation.mutate(payload, {
  onSuccess: () => {
    // 현재 사용자 액션에만 필요한 UI 후처리
  },
});
```

호출부에서는 mutation result 객체를 `{businessAction}Mutation` 이름으로 유지한다.
`mutate`, `isPending`, `isError`, `error`, `isSuccess`, `reset`이 같은 서버
command의 실행 상태라는 점이 드러나야 한다.

```ts
const cancelOrderMutation = useMutation(orderCancelMutations.cancelOrder());

cancelOrderMutation.mutate(payload);
cancelOrderMutation.isPending;
cancelOrderMutation.error;
```

`mutate`만 구조분해해서 업무 함수처럼 alias하는 패턴은 새 코드에서 피한다.
상태를 전혀 쓰지 않는 아주 작은 로컬 예외가 아니라면 아래처럼 쓰지 않는다.

```ts
// Avoid
const { mutate: cancelOrder } = useMutation(orderCancelMutations.cancelOrder());
```

`mutateAsync`는 Promise composition이 필요한 예외 경로다.
GraphQL/Apollo 시절 `async` handler와 `try/catch`로 작성된 로직을 REST +
TanStack Query로 전환할 때, mutation 결과를 `await`해서 다음 비동기 흐름과
조합하지 않는다면 기존 `async` 형태를 유지하지 않는다. 단순 실행, 성공/실패
처리, alert, navigation만 필요한 경우에는 `mutate`를 사용한다.

```ts
// rest-mutation-exception: mutateAsync - 두 mutation 결과를 순차 composition해야 한다
const executeMutation = useMutation(domainMutations.execute());

await executeMutation.mutateAsync(payload);
```

변경된 UI 파일에서 `mutateAsync`를 사유 없이 쓰면 `standards:check`가 실패한다.

## 6. Handler vs Mutation 역할

서버로 가는 비동기 command는 핸들러에 숨기지 않고 mutation으로 상태를 관리한다.

### 핸들러

- 사용자 액션 진입점
- 버튼 클릭, 제출, 확인 등

### mutation

- pending
- success / error
- mutate 호출
- 후처리 연결 지점
- 같은 mutation 객체에서 loading / error / success 상태를 읽는 예측 가능한 surface

즉:

- 핸들러 = 언제 실행할지
- mutation = 실행 상태를 어떻게 관리할지

## 7. 후처리 소유권

### domain에 둘 것

- `mutationFn`
- 필요한 경우에만 `mutationKey`

### 호출부 또는 UI orchestration hook에 둘 것

- alert / toast
- modal / drawer close
- navigation
- pending 이후 UI 정리
- `invalidateQueries`
- `refetch`

query invalidation은 React Query cache orchestration이므로 `api.ts`에 두지 않는다.
호출부에서 수행하되 query key는 domain key factory를 사용하고 문자열을 직접 하드코딩하지 않는다.

### `useMutation` callback과 `mutate` callback 기준

TanStack Query는 mutation 정의 시점의 callback과 `mutate` 호출 시점의 callback을
둘 다 제공한다. 이 프로젝트에서는 둘 중 하나로 강제 통일하지 않고, 후처리의
생명주기에 따라 위치를 나눈다.

#### `useMutation` callback에 둘 것

mutation 성공/실패/완료마다 항상 실행되어야 하는 후처리다.
현재 컴포넌트가 계속 mounted 상태인지와 무관하게 서버 상태 정합성에 필요한
작업은 여기에 둔다.

- `invalidateQueries`
- `refetch`
- React Query cache update
- 같은 mutation을 호출하는 모든 경로에 공통인 success/error/settled 처리

```ts
const releaseReceiptsMutation = useMutation({
  ...receiptAuditMutations.releaseReceipts(auditId),
  onSuccess: () => {
    queryClient.invalidateQueries({
      queryKey: receiptAuditDetailKeys.detail(auditId),
    });
    queryClient.invalidateQueries({ queryKey: unregisteredBarcodeKeys.all });
  },
});
```

#### `mutate` callback에 둘 것

현재 사용자 액션에만 의미가 있고, 호출 지점마다 달라질 수 있는 UI 후처리다.
컴포넌트가 unmount된 뒤에는 실행되지 않아도 되는 작업만 여기에 둔다.

- 현재 modal / drawer 닫기
- 현재 화면의 alert / toast
- 현재 액션 이후 navigation
- 현재 컴포넌트의 local UI reset

```ts
releaseReceiptsMutation.mutate(payload, {
  onSuccess: () => {
    openAlert('반출처리가 완료되었습니다.', {
      onClickPrimary: () => {
        closeAlert();
        onClose();
      },
    });
  },
});
```

하나의 성공 후처리에 cache 정합성과 UI 반응이 모두 있으면, cache 정합성은
`useMutation` callback에 두고 UI 반응은 `mutate` callback에 둔다. 한 화면에서만
쓰이는 mutation이라도 이 기준을 따르면 나중에 호출 경로가 늘어날 때 후처리
누락 위험이 줄어든다.

## 8. Query 갱신 기준

가능한 한 `refetch()` 직접 호출보다 `invalidateQueries()`를 우선한다.
호출 위치는 컴포넌트 또는 UI orchestration hook이다.
query key는 domain key factory를 사용한다.

권장:

```ts
queryClient.invalidateQueries({
  queryKey: domainKeys.lists(),
});
```

예외:

- 특정 query instance를 즉시 재실행해야 하는 특별한 경우

## 9. 커스텀 훅 기준

### 만들지 않는 것

- `useSuspenseQuery(domainQueries.list(...))`만 감싸는 얇은 wrapper hook
- `useMutation(domainMutations.execute())`만 감싸는 compatibility hook

### 허용되는 것

- 여러 query/mutation과 UI 상태 orchestration이 함께 있는 hook
- 화면 문맥 후처리가 반복되어 별도 orchestration hook이 필요한 경우

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

## 10. 테스트 기준

### Query

- queryOptions 자체보다 query를 소비하는 화면/훅의 정책을 테스트
- boundary와 함께 동작하는지 확인

### Mutation

- mutationFn wrapper 자체보다
- 성공/실패 후처리
- invalidate/refetch
- UI 상태 해제
를 테스트

지양:

- React Query 라이브러리 자체가 동작하는지 검증하는 테스트

## 11. 한 줄 기준

이 프로젝트에서 TanStack Query는 이렇게 쓴다.

- 조회는 `useSuspenseQuery`
- 명령은 `useMutation`
- 정의는 domain
- 후처리는 호출부
