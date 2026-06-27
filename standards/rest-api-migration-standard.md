# REST API 마이그레이션 표준

> GraphQL/legacy 호출부를 REST domain API로 전환할 때의 contract, UX, 테스트 기준

- Status: Draft v1
- Date: 2026-04-30
- Scope: GraphQL/legacy API -> REST domain migration

## 1. 목적

이 문서는 기존 GraphQL 응답 구조를 REST 호출부로 옮길 때 프론트가 어떤 기준으로 domain contract와 UX를 설계할지 정한다.

핵심 목표:

1. REST 전환 코드가 GraphQL 응답 관습을 그대로 UX contract로 삼지 않게 한다.
2. command API의 성공/실패 판단을 REST 의미에 맞게 단순화한다.
3. 과도기 백엔드가 기존 wrapper 구조를 유지하더라도 프론트 대응 범위를 최소화한다.
4. caller-facing regression 테스트로 사용자 행동을 보호한다.

## 2. 관련 표준

이 문서는 아래 표준을 전제로 한다.

- `docs/standards/api-domain-ui-standard.md`
  - domain-first 구조
  - generated SDK -> domain/api.ts -> domain/mutations.ts -> UI 흐름
  - UI의 generated SDK 직접 호출 금지
- `docs/standards/tanstack-query-standard.md`
  - command는 `useMutation`
  - `mutationOptions()`는 domain에서 정의
  - alert/refetch/modal close 같은 후처리는 호출부 또는 UI orchestration hook이 소유
- `docs/standards/test-driven-generation-workflow.md`
  - caller migration은 regression-first
  - 테스트 증거는 caller-facing behavior여야 함

## 3. REST 전환 기본 원칙

### 3-1. REST contract를 우선한다

REST 전환의 request/response source of truth 우선순위는 아래와 같다.

1. generated SDK/types와 OpenAPI contract
2. 실제 dev/staging REST 응답 샘플 또는 백엔드 REST 계약 문서
3. domain mapper/type/test fixture
4. 기존 GraphQL operation/type/fixture

기존 GraphQL은 caller 위치와 UI 후처리 확인용 참고자료다.
REST request body wrapper, response envelope, `data` 위치, success/error 판정 기준을
GraphQL shape에서 추정하지 않는다.

REST command API는 기본적으로 아래 의미를 따른다.

| 상황 | 권장 contract | 프론트 처리 |
| --- | --- | --- |
| command 성공 | `2xx` + 성공 body | 성공 후처리 |
| 요청 형식 오류 | `400` | error path |
| 현재 상태와 충돌 | `409` | error path |
| 검증 실패 | `422` | error path |
| 서버 오류 | `5xx` | error path |

프론트는 새 REST caller에서 `successCount`, `failureCount`, `allocationFailed` 같은 GraphQL식 batch result를 기본 UX contract로 삼지 않는다.
다만 REST contract가 명시적으로 batch result를 반환한다면 실제 REST envelope/data path를 그대로 매핑한다.
이때도 GraphQL wrapper를 기준으로 top-level/data 위치를 추정하지 않는다.

### 3-2. 과도기 wrapper 실패는 방어적으로 처리한다

REST 전환 과도기에는 백엔드가 기존 GraphQL wrapper 형태를 그대로 반환할 수 있다.

예:

```ts
{
  success: false,
  message: '재고 할당에 실패했습니다.',
  data: { successCount: 0, failureCount: 1 }
}
```

또는:

```ts
{
  success: true,
  data: { success: true, successCount: 1, failureCount: 1 }
}
```

이 경우 프론트는 아래처럼 최소 대응한다.

1. `success === false`는 실패로 본다.
2. 과도기 body에 `failureCount > 0`이 있으면 실패로 본다.
3. 이 실패 신호를 사용자에게 row별/건별 결과로 보여주지 않는다.
4. domain mutation에서 `throw Error`로 정규화해 UI는 `onError`로 처리한다.

이 대응은 부분 성공 UX를 지원하기 위한 것이 아니라, 과도기 wrapper 실패를 놓치지 않기 위한 임시 호환 계층이다.
부분 실패 자체는 마이그레이션 이후 REST 설계에 맞는 적합한 방식으로 개선할 예정이다. 예를 들어 상태 충돌, 재고 부족, 검증 실패는 `409 Conflict` 또는 `422 Unprocessable Entity` 같은 error response와 명시적인 error body로 표현하고, 프론트는 해당 error path에서 사용자 피드백을 제공하는 방향을 기본으로 한다.

## 4. Batch / Bulk command UX 기준

Bulk command에서 per-row 결과를 보여주려면 응답이 사용자 화면 단위와 1:1로 매핑되어야 한다.

아래 중 하나가 없으면 per-row 성공/실패 UX를 만들지 않는다.

- 화면 row id
- sync product id/code
- order line item id
- 요청 entry와 응답 entry를 안정적으로 연결할 수 있는 client supplied id

응답이 `orderId` 또는 서버 처리 단위만 제공하면, 프론트는 화면 row별 성공/실패를 추정하지 않는다.

권장 UX:

- 성공: 단순 완료 알림
- 실패: 단순 실패/확인 필요 알림
- 후처리: selection 정리, 목록 refetch 또는 query invalidation

지양 UX:

- `N건 성공 / M건 실패`를 화면 row 기준인 것처럼 표시
- `allocationFailed`를 화면 row 실패 목록으로 추정 표시
- 서버 처리 단위 count를 사용자 선택 row count와 섞어 표시

## 5. Domain 구현 기준

### 5-1. API adapter

`domain/api.ts`는 generated SDK 응답을 domain type으로 변환한다.

역할:

- generated SDK 호출
- wrapper error 확인
- DTO -> domain type 변환
- 통신 실패 message 정규화

### 5-2. Mutation option

`domain/mutations.ts`는 command의 caller-facing 성공/실패 contract를 정의한다.

과도기 wrapper 실패가 있는 command는 mutation option에서 실패를 `throw Error`로 정규화한다.

```ts
const result = await executeCommand(params);

if (!result.success || result.failureCount > 0) {
  throw new Error(result.message ?? DEFAULT_FAILURE_MESSAGE);
}
```

UI는 이 mutation을 `useMutation(domainMutations.command())`로 사용하고, `onSuccess`/`onError`에서 후처리를 연결한다.

### 5-3. UI orchestration

UI 또는 orchestration hook은 아래만 소유한다.

- modal/drawer close
- selection clear
- alert
- refetch / invalidate
- navigation
- pending state 연결

UI는 generated SDK 응답 세부 필드를 직접 해석하지 않는다.

## 6. 테스트 기준

마이그레이션 테스트는 caller-facing behavior를 우선한다.

필수 보호 동작:

- 사용자가 액션을 실행하면 올바른 domain mutation params가 전달된다.
- 성공이면 성공 후처리가 실행된다.
- 실패이면 실패 후처리가 실행된다.
- 기존 modal/drawer/selection/refetch 흐름이 유지된다.

보조 contract 테스트:

- 과도기 wrapper 실패를 domain mutation이 error로 변환한다.
- `failureCount > 0`을 부분 성공 UI가 아니라 실패로 정규화한다.

지양 테스트:

- `mutationOptions` 객체가 존재하는지만 확인
- generated SDK wrapper만 직접 호출
- 서버 count를 화면 row count로 맞추는 추정 로직 테스트

## 7. 현재 적용 사례: `stock-item/bulk`

`POST /api/v1/order/commands/stock-item/bulk`는 기존 GraphQL `bulkUpdateOrdersStockItem` 후속 엔드포인트다.

현재 응답은 기존 batch result 구조를 포함하지만, 화면의 선택 row 단위와 서버 처리 단위가 1:1이 아니다.

따라서 이 마이그레이션에서는 아래 기준을 적용한다.

1. 성공/실패 UX만 제공한다.
2. `successCount`, `failureCount`, `allocationFailed`를 row 결과 모달에 사용하지 않는다.
3. `success === false` 또는 `failureCount > 0`은 실패로 정규화한다.
4. 성공 시 "매핑이 변경되었습니다."를 표시한다.
5. 실패 시 backend message 또는 기본 실패 문구를 표시한다.
6. 처리 후 목록을 다시 조회한다.

향후 백엔드가 REST contract를 정리하면 `failureCount` 호환 처리는 제거할 수 있다.

## 8. 마이그레이션 완료 순서와 범위 통제

REST 마이그레이션은 API 단위로 완료한다. 여러 caller를 한 번에 정리하다가 리팩토링 범위가 커지면 완료 기준이 흐려지므로, 아래 순서를 기본 진행 순서로 삼는다.

1. 일반배송/3PL 일반배송 송장출력
   - `invoiceControllerRegisterGoodsFlowCommonOrder`
   - `invoiceControllerRegisterGoodsFlow3PlOrder`
2. 매핑 수정 계열
   - `orderCommandControllerUpdateOrderStockItem`
   - `orderCommandControllerUpdateOrderGoodsItem`
   - `orderCommandControllerUpdateOrderSyncProduct`
3. OutOfStock 대체상품 일괄 연결
   - `orderCommandControllerBulkUpdateOrderStockItem`
4. 단일 버튼형 command
   - `invoiceControllerCancelCjDawnDelivery`
   - `orderCommandControllerUpdateOrdersReserveDate`
   - `orderCommandControllerUpdateOrdersDeliveryMethod`
   - `orderCommandControllerUpdateOrderWorkSettings`
5. 주문 취소 계열
   - `orderCommandControllerCancelOrder`
   - `orderCommandControllerCancelConfirmOrder`
   - `orderCommandControllerCancelOrdersByOrderIds`
   - `orderCommandControllerCancelOrdersToConfirmed`
   - `orderCommandControllerRollbackOrderStatusToLocked`
6. 상태전이/주문확정/출고완료 계열
   - `orderCommandControllerUpdateOrdersStatus`
   - `orderCommandControllerUpdateOrdersToShipmentComplete`
   - `orderCommandControllerUpdateInvoiceNumbersAndConfirm`
   - `orderCommandControllerUpdateInvoiceNumbersForAnyStatus`

진행 규칙:

- 하나의 커밋은 가능한 한 위 순서의 한 묶음만 완료한다.
- `implemented` 판정은 domain adapter, caller migration, 회귀 테스트 또는 명시적 검증 근거가 함께 있을 때만 완료로 본다.
- 진행 중 발견한 리팩토링은 현재 API 완료를 막는 문제가 아니면 후순위 작업으로 기록한다.
- 후순위 리팩토링이 필요해도 현재 API 완료 순서를 바꾸지 않는다.
- 공통화가 필요하면 먼저 API 단위 완료를 끝내고, 별도 리팩토링 커밋에서 다룬다.
- 마이그레이션 계획서의 frontmatter가 `status: completed`가 되면 reminder hook은 다음 `git commit` 이후 제거된다.

이 규칙은 리팩토링을 금지하기 위한 것이 아니다. 마이그레이션 완료 단위를 작게 유지해 검토와 배포 리스크를 낮추기 위한 진행 가드다.

## 9. 마이그레이션 체크리스트

- [ ] 이 API가 query인지 command인지 정의했는가
- [ ] request/response source of truth가 generated/OpenAPI/실제 REST 샘플인지 확인했는가
- [ ] REST response envelope과 domain result data path를 확인했는가
- [ ] domain aggregate/capability가 endpoint명이 아니라 업무 언어 기준인가
- [ ] UI가 generated SDK를 직접 호출하지 않는가
- [ ] command 성공/실패를 REST 의미로 해석했는가
- [ ] GraphQL식 batch result를 UX contract로 삼지 않았는가
- [ ] mapper/test fixture가 GraphQL response shape를 REST fixture로 둔갑시키지 않았는가
- [ ] per-row 결과를 표시한다면 응답에 안정적인 row 식별자가 있는가
- [ ] 과도기 `success:false` / `failureCount > 0` wrapper 실패를 놓치지 않는가
- [ ] alert/refetch/modal/selection 후처리가 보존되었는가
- [ ] caller-facing regression 테스트가 있는가
- [ ] 남은 과도기 호환 로직에 제거 조건을 설명했는가
- [ ] 새 리팩토링 아이디어를 현재 API 완료 범위에 끌어들이지 않고 후순위로 기록했는가
- [ ] envelope 풀이를 `classifyEnvelope` 단일 진입점에 가뒀는가 (§10)

## 10. Envelope classifier 사용 규칙

### 10-1. 책임 분리

OMS 응답 envelope 풀이는 단일 함수 `classifyEnvelope`(`src/shared/outcome/`)에 가둔다.
도메인 mapper / api / mutation hook / UI는 SDK `{ data, error }` shape와 envelope shape를 직접 보지 않는다.

| layer | 보는 것 | 결정하는 것 |
| --- | --- | --- |
| `classifyEnvelope` | SDK `{ data, error }` | envelope 풀이, ok/err 분류, 실패면 `DomainOutcomeError` throw |
| 도메인 mapper / api | `{ data, message }` 또는 throw | ok→도메인 모델 변환, 실패는 그대로 전파 |
| mutation hook | `Promise<TDomain>` 또는 throw | `onSuccess(value)` / `onError(message)` |
| UI | 도메인 outcome | 화면 표현 |

### 10-2. 도메인 mapper / api 표준 모양

```ts
import { classifyEnvelope } from '@shared/outcome/envelopeClassifier';

interface StockAllocationDataDto {
  isSuccess?: boolean;
  successOrderIds?: string[];
  failedOrders?: { orderId?: string; reason?: string }[];
}

export const updateOrderStockAllocationStatus = async (
  params: UpdateOrderStockAllocationStatusParams,
): Promise<string> => {
  const sdkResult = await sdkCall({ client: omsClient, body: params });

  const { message } = classifyEnvelope<StockAllocationDataDto>(sdkResult, {
    fallbackErrorMessage: '재고 할당 상태 변경에 실패했습니다.',
    detectBusinessFailure: (data) => {
      const failed = data?.failedOrders?.length ?? 0;
      if (data?.isSuccess === true && failed === 0) return null;
      return {
        status: 409,
        detail: data?.failedOrders?.[0]?.reason ?? '재고 할당 상태 변경에 실패했습니다.',
      };
    },
  });

  return message ?? '재고 할당 상태가 변경되었습니다.';
};
```

실패 경로는 `classifyEnvelope`이 `DomainOutcomeError`를 throw하므로 도메인이 별도 분기를 가질 필요가 없다.

### 10-3. detectBusinessFailure 컨벤션

envelope.data 내부의 비즈니스 실패 신호(`isSuccess`, `failedOrders[]`, `successCount`/`failureCount` 등)는
도메인이 `detectBusinessFailure` predicate으로 분류 함수에 주입한다.

- predicate은 **envelope-내부 신호에만 의존**해야 한다. 화면 상태, selection, 외부 store에 의존하면 단일 진입점 책임이 깨진다.
- 합성 status (예: 409, 422)는 도메인이 의미를 부여한다. 백엔드 envelope.code 카탈로그가 안정화되면 전역 매핑으로 끌어올린다.
- detectBusinessFailure가 인라인으로 길어지면 도메인 폴더 안에 별도 파일로 분리한다 (예: `domain/foo/businessFailure.ts`).

### 10-4. envelope 진입점 단일화

`classifyEnvelope` 외에서 envelope shape (`success` / `code` / `message` / `data`)을 직접 읽지 않는다.
도메인 코드가 envelope 필드를 직접 보고 있으면 컨벤션이 바뀔 때 일괄 교체가 어려워진다 — PR 리뷰에서 잡는다.

- mutation hook의 `onError(error)`는 `DomainOutcomeError.message`를 그대로 받으며, hook이 envelope을 다시 풀지 않는다.
- 도메인 mapper가 `sdkResult.data`/`sdkResult.error`를 직접 분해하지 않는다.

### 10-5. 테스트 보호막

- `classifyEnvelope` 단위 테스트는 `src/shared/outcome/__tests__/`에서 ok / transport error / wrapper 실패 / 응답 형식 오류 / detectBusinessFailure / 우선순위 케이스를 망라한다.
- 도메인별 mapper unit test는 `classifyEnvelope`을 사용한 채로 fixture를 envelope-aware로 작성한다.
- caller-facing regression test (vitest + RTL)는 success/failure 두 경로의 alert/modal/refetch 동작을 보존한다.

### 10-6. 백엔드 응답 컨벤션 전환 시 처리

백엔드가 envelope을 제거하고 HTTP status 정공 설계(예: RFC 9457 Problem Details)로 전환하면:

1. `classifyEnvelope` 모듈을 deprecate하고 더 단순한 helper (예: `assertSdkOk`)로 교체하거나 도메인 mapper에서 인라인 분해한다.
2. 도메인별 `detectBusinessFailure` predicate은 HTTP status가 같은 역할을 흡수하므로 제거한다.
3. 합성 status 매핑은 진짜 status를 그대로 사용한다.

전환 PR은 mechanical change에 가까워야 한다 — envelope 풀이가 단일 진입점에 모여 있어 호출부 grep 한 번으로 모두 교체 가능. 이게 안 되면 도메인이 §10-4 단일 진입점 규칙을 위반한 신호다.
