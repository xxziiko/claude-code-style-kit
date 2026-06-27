# scaffold:rest-domain 설계

> REST 표준화 작업에서 domain/query/mutation 기본 구조를 표준 템플릿으로 생성하기 위한 스캐폴더 설계 문서

- Status: Draft
- Date: 2026-04-24
- Scope: Repo-wide

## 1. 목적

이 문서는 `scaffold:rest-domain`이 어떤 입력을 받고,
어떤 파일을 생성하며,
어디까지 자동화하고 어디서 멈춰야 하는지 정의한다.

목표:

1. reference domain 패턴을 반복 가능한 템플릿으로 만든다.
2. boilerplate를 줄이되, 비즈니스 판단은 자동화하지 않는다.
3. `codegen:rest` 이후 표준 도메인 골격을 빠르게 만들 수 있게 한다.

## 2. 전제

이 스캐폴더는 아래를 전제로 한다.

1. generated SDK/types는 이미 존재한다.
2. runtime contracts가 필요하면 이미 생성되었거나 생성 가능한 상태다.
3. 도메인 경계는 사람이 먼저 판단한다.
4. aggregate / capability는 proposal과 사용자 승인으로 먼저 확정한다.

즉, 이 도구는 도메인을 “판단”하지 않고, 도메인을 “생성”한다.
`command`, `mutation`, `api`, `rest` 같은 기술 용어는 domain 이름으로 받지 않는다.

## 3. 커맨드 목표

예상 명령:

```txt
yarn scaffold:rest-domain --aggregate <aggregate> --capability <capability> --service <wms|oms> ...
```

또는

```txt
node scripts/scaffold-rest-domain.mjs --aggregate <aggregate> --capability <capability> --service <wms|oms> ...
```

Compatibility:

```txt
yarn scaffold:rest-domain --domain <aggregate>/<capability> --service <wms|oms> ...
```

## 4. 입력값

### 필수 입력

- `--aggregate`
  - 생성할 domain aggregate 이름
  - 예: `stock`, `order`, `admin`

- `--capability`
  - aggregate 아래의 업무 capability 이름
  - 예: `expiration`, `overview`, `featureFlag`

- `--mode`
  - 도메인 성격
  - 후보:
    - `query`
    - `command`
    - `workflow`

- `--service`
  - generated SDK service 이름
  - 후보:
    - `wms`
    - `oms`

### 선택 입력

현재 스캐폴더는 아래 입력만 받는다.

```txt
--aggregate <aggregate>
--capability <capability>
--domain <aggregate>/<capability>  # compatibility alias for aggregate/capability
--mode <query|command|workflow>
--service <wms|oms>
--query <generated-operation>      # optional
--mutation <generated-operation>   # optional, repeatable
--with-dto                         # optional
--with-mapper                      # optional, implies --with-dto
--with-errors                      # optional
--with-rules                       # optional
--force                            # optional
--dry-run                          # optional
```

## 5. 출력 파일

스캐폴더는 승인된 manifest에 포함된 파일만 생성한다.

허용 파일명:

```txt
src/domain/{aggregate}/{capability}/
  index.ts
  api.ts
  types.ts
  dto.ts            # DTO boundary가 필요할 때
  queries.ts        # mode에 따라
  mutations.ts      # mode에 따라
  mappers.ts        # DTO -> query 변환이 필요할 때
  errors.ts         # 필요 시
  rules.ts          # 필요 시
```

위 목록 밖의 파일이 필요하면 스캐폴더 실행 전에 manifest를 먼저 갱신한다.

### mode별 기본 생성 규칙

#### `query`

- `api.ts`
- `types.ts`
- `dto.ts`
- `mappers.ts`
- `queries.ts`

#### `command`

- `api.ts`
- `types.ts`
- `mutations.ts`

#### `workflow`

- `api.ts`
- `types.ts`
- `queries.ts`      # --query가 있을 때만
- `mutations.ts`    # --mutation이 있을 때만
- `dto.ts`, `mappers.ts` # --query가 있을 때

### mode별 필수 조건

#### `query`

- `--query` 필수
- `--mutation` 없음

#### `command`

- `--mutation` 1개 이상 필수
- `--query` 없음

#### `workflow`

- `--query` 또는 `--mutation` 둘 다 가능
- 일반적으로 둘 다 있는 경우에 적합

## 6. 생성되는 템플릿 수준

자동 생성은 아래까지만 한다.

### `api.ts`

- generated SDK import placeholder
- function signature
- command/query 구분된 기본 함수
- TODO 주석

예상 골격:

```ts
// generated import placeholder
// domain types import

export const fetchDomainList = async (...) => {
  // TODO: generated SDK 연결
  // TODO: DTO -> domain 변환
};
```

### `types.ts`

- capability 조회 결과/query 타입과 API 호출 경계 params/payload/result 타입
- UI 표현/상호작용 상태는 넣지 않는다.

예상 골격:

```ts
export interface DomainListParams {}
export interface DomainItem {}
export interface DomainList {}
```

### `queries.ts`

- query key factory
- `queryOptions()` 기본 골격

예상 골격:

```ts
export const domainKeys = {
  all: ['domain'] as const,
  lists: () => [...domainKeys.all, 'list'] as const,
  list: (params: DomainListParams) => [...domainKeys.lists(), params] as const,
};

export const domainQueries = {
  list: (params: DomainListParams) =>
    queryOptions({
      queryKey: domainKeys.list(params),
      queryFn: () => fetchDomainList(params),
    }),
};
```

### `mutations.ts`

- `mutationOptions` factory 기본 골격
- `mutationFn` 중심 구현
- `mutationKey`는 선택 사항

예상 골격:

```ts
export const domainMutations = {
  create: () =>
    mutationOptions({
      mutationFn: (payload: DomainCreateParams) => createDomain(payload),
    }),
};
```

`mutationKey`는 mutation identity를 실제로 소비하는 경우에만 추가한다.
예: `useMutationState`, `isMutating`, mutation defaults, 공통 pending 추적.
단순 command 실행과 invalidate에는 필요하지 않다.
query invalidation이 필요하면 호출부 또는 UI orchestration hook의 `onSuccess`에서 처리한다.
단, `api.ts`에는 `queryClient`를 두지 않는다.

### concrete business rule/value files

- smart constructor 골격
- `isXxx`
- `createXxx`
- `UnsupportedXxxError`

필요하면 `expirationDate.ts`, `stockQuantity.ts`, `dateRange.ts`처럼 구체적인 비즈니스 이름을 사용한다.
이 파일은 manifest에 승인된 경우에만 생성한다.

## 7. 자동화가 하지 않을 것

아래는 스캐폴더가 자동으로 결정하지 않는다.

1. 이 API가 정말 이 domain에 속하는지
2. 이 API가 어떤 aggregate / capability에 속하는지
3. generated operation이 query인지 command인지
4. 어떤 query를 invalidate해야 하는지
5. value object 이름의 최종 의미
6. category / builder / rule 매핑
7. orchestration hook 내용

또한 아래도 자동으로 하지 않는다.

8. `invalidateQueries` 대상 결정 없이 임시 invalidation을 생성
9. `useSuspenseQuery` 예외 판단
10. generated runtime contracts 대상 확정

## 8. reference domain과의 관계

스캐폴더는 아래 기준 도메인의 패턴을 복제 가능해야 한다.

### history / query

- `receiptHistory`

### command / action

- `packing`

즉, 스캐폴더는 이 두 reference를 일반화한 템플릿이어야 한다.

## 9. 생성 후 에이전트가 기본적으로 채워야 할 일

스캐폴더 생성 후에는 승인된 입력과 표준으로 이미 결정 가능한 항목을 에이전트가 먼저 채운다.

1. generated SDK 연결
2. query operation과 API boundary 타입 surface 구현
3. DTO -> domain 변환 구현
4. 승인된 invalidate key 반영
5. 의미 있는 테스트 설계를 위한 입력 정리

아래는 여전히 사람 승인/판단이 필요한 비결정적 항목이다.

1. query 이름 재판단
2. concrete business rule/value file 필요 여부 재판단
3. 표준 밖의 후처리 / invalidation 예외
4. 추가 비즈니스 테스트 시나리오 확장

## 10. 생성 파일별 TODO 기준

스캐폴더는 TODO를 명시적으로 남겨야 한다.

### `api.ts`

- TODO: generated SDK 연결
- TODO: DTO -> domain 변환 구현
- TODO: 에러 정규화 기준 반영

### `types.ts`

- TODO: API 호출 경계 params/payload/result 확정
- TODO: capability query 이름과 필드 확정
- TODO: UI interaction/display type 혼입 금지

### `queries.ts`

- TODO: staleTime/gcTime 결정
- TODO: query key 세부 정책 보강

### `mutations.ts`

- TODO: mutation 이름 확정
- TODO: command input/output 타입 연결
- TODO: 호출부 후처리 책임 문서화

## 11. 안전장치

스캐폴더는 아래 안전장치를 가져야 한다.

1. 기존 파일이 있으면 덮어쓰기 전에 중단 또는 명시적 `--force` 요구
2. mode와 입력이 맞지 않으면 실패
3. manifest 밖 신규 파일 생성 금지
4. 생성 후 요약 출력
5. reference 문서 링크 출력
6. 결정적 구현 후 self-check checklist 실행

## 12. 생성 후 출력 요약

예상 출력:

```txt
Generated:
- src/domain/wms/packing/index.ts
- src/domain/wms/packing/api.ts
- src/domain/wms/packing/types.ts
- src/domain/wms/packing/mutations.ts

Omitted by design:
- queries.ts: no query operation
- errors.ts: no approved error contract
- rules.ts: no approved pure domain rule
- feature files: use connect-api after domain scaffold verification
```

## 13. 성공 기준

이 스캐폴더가 성공적이라고 보려면:

1. 생성된 파일 구조가 표준과 일치한다.
2. query / mutation 기본 패턴이 자동으로 깔린다.
3. generated -> domain -> UI 경계가 자연스럽게 유지된다.
4. 사람이 결정해야 하는 영역은 자동화가 침범하지 않는다.

## 14. 1차 구현 범위

1차 구현에서는 아래만 목표로 한다.

1. `--aggregate`
2. `--capability`
3. `--mode`
4. `--query`
5. `--mutation`

즉, 먼저 작은 골격 생성기로 시작하고,
runtime contracts / value object 자동 연결은 이후 확장한다.

## 15. 한 줄 정의

`scaffold:rest-domain`은 "표준 도메인 골격 생성기"이지, "도메인 설계기"가 아니다.
