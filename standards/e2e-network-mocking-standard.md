# E2E Network Mocking Standard

> Playwright E2E와 MSW browser worker의 네트워크 mocking 책임을 분리하는 표준

- Status: Draft v2
- Date: 2026-05-29
- Scope: Playwright E2E, MSW browser worker, local/demo mock app flows

## 1. 목적

이 문서는 브라우저 기반 검증에서 API mock의 소유권을 명확히 나눈다.

핵심 목표:

1. Automated E2E에서 test별 request/response 제어를 안정화한다.
2. MSW browser worker를 로컬 개발/데모/mock app 용도로 명확히 둔다.
3. Playwright `page.route()`와 MSW browser worker가 같은 요청을 동시에 소유하지 않게 한다.
4. 실패 원인이 UI regression인지, mock ownership 충돌인지 빠르게 분리한다.

## 2. 표준 모드

`yarn dev`는 `.env.develop`의 `VITE_ENABLE_MSW=true` + `VITE_MSW_TARGET=preview` 설정으로 MSW preview target을 자동 활성한다. 이 상태에서 두 가지 검증 모드의 ownership은 다른 명령이 아니라 **Service Worker 차단 메커니즘**으로 구분한다.

### 2-1. Automated E2E

Automated Playwright E2E는 아래 조합을 사용한다.

```text
yarn dev + Playwright page.route() + serviceWorkers: 'block'
```

역할:

- Vite dev server는 실제 앱 bundle과 routing을 제공한다.
- Playwright project 설정의 `serviceWorkers: 'block'`이 MSW browser worker 등록을 차단한다. 이 차단이 ownership boundary이다.
- 각 Playwright spec은 `page.route()`로 필요한 REST/GraphQL 응답을 직접 소유한다.
- success, `success:false`, 4xx/5xx, timeout 같은 응답 변형은 spec 안에서 명시한다.
- request body assertion이 필요한 경우 spec의 route handler에서 body를 capture한다.

2026-05-08 SLI-885 note: `serviceWorkers: 'block'` 제거 후 focused order E2E(`order-pending`, `shipment-complete`, `update-delivery-date`) 10개 중 6개가 실패했다. 따라서 현재 automated order E2E lane에서는 이 설정을 유지한다.

### 2-2. Local / Demo Mock App

수동 검증, 데모, exploratory QA는 `yarn dev`를 그대로 사용한다.

```text
yarn dev + MSW browser worker (preview target)
```

역할:

- `.env.develop`의 `VITE_ENABLE_MSW=true` + `VITE_MSW_TARGET=preview`로 preview target에 등록된 MSW handler만 활성화된다.
- 현재 preview target에는 `stockHandlers`만 등록되어 있다. 나머지 도메인 요청은 unhandled로 처리되어 실제 dev 백엔드로 빠진다.
- 백엔드 endpoint가 아직 없는 도메인 화면을 mock 데이터로 수동 검증할 수 있다. 예: SLI-994 stock 화면.
- Playwright spec별 `page.route()` override는 기본적으로 사용하지 않는다.


## 3. 금지 / 예외

### 3-1. 기본 금지

Automated E2E에서 MSW browser worker와 Playwright `page.route()`가 같은 API 요청을 동시에 소유하지 않는다.

금지 예:

- automated E2E project에서 `serviceWorkers: 'block'`을 해제한다 (MSW worker가 등록되면서 spec의 `page.route()` handler와 소유권이 충돌).
- 앞의 금지를 어긴 패턴: `page.route()`와 MSW handler가 같은 `/api/v1/...` URL을 동시에 접수한다.

### 3-2. 예외

아래 조건을 모두 만족할 때만 혼합을 허용한다.

1. spec 또는 helper 주석에 ownership boundary를 명시한다.
2. MSW와 `page.route()`가 같은 URL pattern을 소유하지 않는다.
3. 테스트 실패 시 어떤 layer가 요청을 처리했는지 확인할 수 있는 evidence를 남긴다.

예외 예:

- MSW browser worker가 앱의 내부 API를 소유하고, Playwright `page.route()`는 외부 CDN asset 또는 analytics endpoint만 차단한다.
- Playwright가 특정 third-party callback만 mock하고, 앱 API는 MSW scenario가 소유한다.
- 특정 browser capability 테스트에서 service worker가 필요하고, API mock은 충돌하지 않는 URL boundary로 분리한다.

## 4. 선택 기준

| 목적 | 표준 선택 | 이유 |
| --- | --- | --- |
| 사용자 workflow regression 자동화 | `yarn dev` + `page.route()` + `serviceWorkers: 'block'` | test별 응답과 request body를 결정적으로 제어한다. |
| REST migration caller behavior 검증 | `yarn dev` + `page.route()` + `serviceWorkers: 'block'` | success/failure/business failure를 spec별로 분리한다. |
| 백엔드 없이 로컬 화면 수동 확인 | `yarn dev` + MSW browser worker (preview) | preview target의 mock backend 위에서 stock 도메인을 자연스럽게 확인한다. |
| 데모/PR preview mock app | `yarn dev` + MSW browser worker (preview) | spec이 아니라 환경이 mock scenario를 소유한다. |
| 실제 서버 연결 확인 | staging/live smoke | mock은 connectivity 증거가 아니다. |

## 5. 작성 규칙

Playwright E2E spec은 아래를 따른다.

1. API 응답이 테스트 판단에 중요하면 spec 또는 가까운 helper에서 `page.route()`로 명시한다.
2. 성공/실패/비즈니스 실패는 서로 다른 test나 명시적인 setup helper로 분리한다.
3. 사용자 관찰 가능 결과는 browser native dialog event가 아니라 DOM role/text로 검증한다.
4. request body가 migration gate이면 route handler에서 capture하고 assertion한다.
5. MSW browser worker에 의존하는 E2E spec은 파일 상단에 이유와 mock ownership boundary를 적는다 (예외 케이스).

## 6. 현재 프로젝트 적용

현재 기본 Playwright `chromium` project는 automated E2E lane으로 본다.

```text
webServer.command = yarn dev
API mocks = Playwright page.route()
use.serviceWorkers = block
```

`yarn dev`가 `.env.develop`을 통해 MSW preview를 자동 등록하는 설정이지만, automated E2E lane에서는 `serviceWorkers: 'block'`이 worker 등록을 차단해 `page.route()` 소유권을 보존한다.

MSW browser worker 또는 service worker 자체를 검증하는 E2E가 필요하면 별도 Playwright project를 추가하고, 그 project에서는 service worker 차단을 사용하지 않는다.
