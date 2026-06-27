# E2E 테스트 & CI 전략

> Sirloin OMS Front E2E 테스트 운영 가이드

이 문서는 **현재 저장소에 존재하는 Playwright 테스트 구조**, **GraphQL/REST 혼합 mock 전략**, **CI 선택 실행 방식**을 기준으로 정리합니다.

---

## 1. 테스트 계층 구조

```txt
     /  E2E  \          ← 크리티컬 유저 저니 (Playwright)
    /─────────\
   /   통합    \        ← 컴포넌트 + 훅 (Vitest + RTL + MSW)
  /─────────────\
 /   단위 테스트  \      ← 유틸, 도메인 로직 (Vitest)
```

| 계층 | 도구 | 위치 | 실행 |
|------|------|------|------|
| Unit | Vitest | `src/**/__tests__/` | `yarn test` |
| Integration | Vitest + RTL + MSW | `src/**/__tests__/` | `yarn test` |
| E2E | Playwright | `e2e/` | `yarn test:e2e` |

---

## 2. 현재 E2E 구조

```txt
e2e/
├── .env.test
├── auth/
│   ├── auth.setup.ts
│   └── login.spec.ts
├── helpers/
│   ├── graphql-mock.ts
│   └── rest-mock.ts
├── order/
│   ├── goodsflow-invoice.spec.ts
│   ├── order-all.spec.ts
│   └── out-of-stock-order.spec.ts
└── wms/
    ├── receipt-group.spec.ts
    ├── receipt-history-ripening.spec.ts
    ├── stock-inquiry-overall.spec.ts
    ├── stock-item-modify-ripening.spec.ts
    ├── stock-operation.spec.ts
    ├── stock-status-detail.spec.ts
    ├── stock-status-packing.spec.ts
    ├── stock-status-pre-packing.spec.ts
    └── stock-status-sales-deadline.spec.ts
```

### helper 역할

- `e2e/helpers/graphql-mock.ts`
  - GraphQL 엔드포인트 intercept helper
  - loginCheck / login / 기존 GraphQL query mock에 사용
- `e2e/helpers/rest-mock.ts`
  - REST 엔드포인트 intercept helper
  - 재고실사, 재고 상태 상세 등 REST 기반 시나리오에 사용

즉, 이 저장소의 E2E는 **GraphQL helper와 REST helper가 공존하는 혼합 전략**입니다.

---

## 3. Playwright 프로젝트 실행 순서

```txt
setup (auth.setup.ts)
  → chromium (인증 필요 테스트)
    → login-tests (로그인 페이지 전용)
```

- `setup`: 실제 로그인 후 storageState 저장
- `chromium`: 저장된 인증 상태 재사용
- `login-tests`: 인증 없이 로그인 UI 검증

---

## 4. 테스트 계층: smoke vs QA 시나리오

### 계층 정의

| 계층 | 태그 | 정의 | 실행 시점 |
|------|------|------|-----------|
| Smoke | `@smoke` | "페이지가 살아있는가" — 라우팅 + 초기 렌더링 + 기본 데이터 표시 확인 | PR 선택 실행, nightly |
| QA 시나리오 | `@order` / `@wms` (no `@smoke`) | 특정 API 응답 케이스별 모달 상태, 비즈니스 플로우 검증 | nightly만 |

### @smoke 포함 기준

다음 조건을 **모두** 만족해야 `@smoke` 태그를 부여합니다.

- "이 페이지가 로드되고 기본 데이터가 표시되는가"만 검증
- 특정 비즈니스 시나리오(복수 API 응답 케이스, 모달 상태, 플로우 분기)를 검증하지 않음
- 빠른 시그널이 목적이며 30초 내 완료 가능

### @smoke 제외 기준 (QA 시나리오)

다음 중 하나라도 해당하면 `@smoke` 없이 도메인 태그만 부여합니다.

- 특정 출고제한, 재고할당 실패 등 API 응답 케이스를 mock해서 검증
- 모달 open/close, refetch 여부 등 상태 전이를 검증
- 여러 배송 유형 분기 등 비즈니스 플로우 특화 시나리오

### 현재 smoke 테스트 목록

| 파일 | 태그 | 비고 |
|------|------|------|
| `auth/login.spec.ts` | `@auth @smoke` | 유지 |
| `order/order-all.spec.ts` | `@order @smoke` | 유지 |
| `wms/stock-inquiry-overall.spec.ts` | `@wms @smoke` | 유지 |

> `order/shipment-restricted-smoke.spec.ts`는 출고제한 비즈니스 시나리오 특화이므로 `@smoke` 제거 대상입니다.

### 태깅 시스템

| 태그 | 의미 | 용도 |
|------|------|------|
| `@order` | 주문 도메인 | 주문 관련 변경 시 선택 실행 |
| `@wms` | WMS 도메인 | WMS 관련 변경 시 선택 실행 |
| `@auth` | 인증 도메인 | 인증 관련 변경 시 선택 실행 |
| `@smoke` | 페이지 생존 확인 | PR 선택 실행 대상 |

### 태그 예시

```typescript
// smoke: 페이지 로드만 확인
test.describe('재고 현황 페이지', { tag: ['@wms', '@smoke'] }, () => {
  test('기본 목록을 렌더링한다', async ({ page }) => {
    // ...
  });
});

// QA 시나리오: 비즈니스 플로우 검증 (nightly only)
test.describe('출고제한 결과 모달', { tag: ['@order'] }, () => {
  test('일부 성공 + 출고제한 항목이 표시된다', async ({ page }) => {
    // ...
  });
});
```

### 실행 명령어

```bash
yarn test:e2e
yarn test:e2e:smoke
yarn test:e2e:order
yarn test:e2e:wms

# 여러 태그 조합
yarn test:e2e -- --grep "@order|@wms"
```

---

## 5. 로컬 실행 가이드

### 사전 준비

```bash
# 1. Playwright 브라우저 설치 (최초 1회)
npx playwright install chromium

# 2. 테스트 계정 확인
cat e2e/.env.test
# E2E_TEST_EMAIL=...
# E2E_TEST_PASSWORD=...

# 3. 개발 서버 실행
yarn dev
# → http://localhost:3000
```

### 자주 쓰는 명령어

```bash
yarn test:e2e
yarn test:e2e:smoke
yarn test:e2e:ui
yarn test:e2e:debug
yarn test:e2e:headed
yarn test:e2e:report
```

---

## 6. Mock 전략

## 6-1. GraphQL mock 사용 시점

다음 상황에서는 `graphql-mock.ts`를 우선 사용합니다.

- 기존 Apollo/GraphQL 기능 검증
- operationName 기반으로 응답을 분기해야 할 때
- loginCheck / login 같은 인증 GraphQL 흐름

예시:
- `mockLoginCheck(page)`
- `mockAllStockList(page, mockData)`

## 6-2. REST mock 사용 시점

다음 상황에서는 `rest-mock.ts`를 우선 사용합니다.

- React Query + REST 기반 페이지/도메인 검증
- `**/api/v1/...` 경로를 직접 intercept해야 할 때
- generated REST SDK를 사용하는 흐름 검증

예시:
- 재고실사 관련 API 전체 mock
- 재고 상태 주문/receipt 상세 mock

## 6-3. 선택 원칙

- **GraphQL 엔드포인트를 호출하는 기능** → GraphQL mock
- **REST 엔드포인트를 호출하는 기능** → REST mock
- 한 시나리오에서 둘 다 쓰면 helper를 함께 조합 가능

---

## 7. 새 E2E 테스트 작성 가이드

### 체크리스트

1. 적절한 도메인 디렉토리에 `*.spec.ts` 생성
2. `test.describe()`에 도메인 태그 부여 (`@order`, `@wms`, `@auth`)
3. "페이지가 살아있는가" 수준이면 `@smoke` 태그 추가 — 비즈니스 시나리오 특화면 도메인 태그만
4. 사용하는 데이터 페칭 스택에 맞는 helper 선택
   - GraphQL → `graphql-mock.ts`
   - REST → `rest-mock.ts`
5. 인증 필요 테스트는 `mockLoginCheck(page)` 또는 auth setup 재사용

### 기본 템플릿

```typescript
import { expect, test } from '@playwright/test';
import { mockLoginCheck } from '../helpers/graphql-mock';

test.describe('페이지명', { tag: ['@wms'] }, () => {
  test.beforeEach(async ({ page }) => {
    await mockLoginCheck(page);
  });

  test('정상 로드', async ({ page }) => {
    await page.goto('/경로');
    await expect(page).toHaveURL(/\/경로/);
  });
});
```

### GraphQL mock 예시

```typescript
await page.route('**/wms/graphql', async (route) => {
  const body = route.request().postDataJSON() as { operationName?: string } | null;
  if (body?.operationName === 'myQuery') {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ data: mockData }),
    });
  } else {
    await route.fallback();
  }
});
```

### REST mock 예시

```typescript
await page.route('**/api/v1/stock-item**', async (route) => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify({
      success: true,
      data: {
        totalCount: 1,
        totalPage: 1,
        currentPage: 1,
        stockItems: [],
      },
    }),
  });
});
```

---

## 8. CI 선택 실행 전략

`scripts/detect-e2e-domains.sh`가 변경 파일을 분석하여 실행 범위를 정합니다.

현재 의미 있는 규칙 예시:

| 변경 파일 | 실행 범위 |
|-----------|-----------|
| `src/pages/order/**` | `@order` |
| `src/hooks/wms/**` | `@wms` |
| `src/features/stock/**` | `@wms` |
| `src/features/order/**` | `@order` |
| `src/features/auth/**` | `@auth` |
| `src/features/{unknown}/**` | 전체 |
| `src/api/generated/wms/**` | `@wms` |
| `src/api/generated/oms/**` | `@order` |
| `src/api/**` | 전체 |
| `src/shared/**` | 전체 |
| `src/components/shared/**` | 전체 |
| `src/domain/shared/**` | 전체 |
| `src/domain/stock/**` | `@wms` |
| `src/domain/order/**` | `@order` |
| `src/domain/{unknown}/**` | 전체 |
| `e2e/helpers/*` | importing spec 기준으로 역추적 |

즉, GraphQL generated 파일뿐 아니라 **REST generated SDK 변경도 선택 실행 판단에 포함**됩니다.

---

## 9. Flaky 대응 원칙

### 자주 있는 원인

| 증상 | 원인 | 해결 |
|------|------|------|
| `timeout waiting for selector` | 데이터 로딩 지연 | mock 사용, 응답 대기 추가 |
| `element detached from DOM` | 응답/리렌더 timing 문제 | `Promise.all + waitForResponse` 패턴 |
| `strict mode violation` | selector가 너무 넓음 | 더 구체적 locator 사용 |
| 간헐적 실패 | race condition | 클릭/이동 전에 응답 대기 설정 |

### 추천 패턴

```typescript
await Promise.all([
  page.waitForResponse(isOutOfStockOrdersResponse),
  page.goto('/order/out-of-stock'),
]);
```

---

## 10. 운영 원칙 요약

- PR에서는 **변경 기반 선택 실행**으로 빠른 피드백을 얻음
- Nightly에서는 **전체 suite 실행**으로 크로스 도메인 회귀를 잡음
- 새 테스트 작성 시 가장 먼저 **이 기능이 GraphQL인지 REST인지**를 판단해야 함
- `@smoke`는 "페이지가 살아있는가" 수준만 — 비즈니스 플로우 검증은 QA 시나리오(`@order`/`@wms`)로 분리
- QA 시나리오 테스트는 PR 선택 실행에서 트리거되지 않으며, nightly 전체 실행에만 포함됨
