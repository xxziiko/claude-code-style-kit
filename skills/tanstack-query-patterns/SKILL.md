---
name: tanstack-query-patterns
description: TanStack Query, domain mapper, query/mutation factory, query key, invalidation, 서버 상태 훅을 구현하거나 검토할 때 사용. 프로젝트 의존 API adoption workflow 없이 query/mutation scaffold와 경계를 점검한다.
allowed-tools: Read, Grep, Glob, Edit, Write
---

# TanStack Query 규칙 라우터

이 스킬은 TanStack Query 작업을 표준 문서와 경계 점검으로 연결하는 라우팅 문서다.
query/mutation scaffold, cache key, invalidation, DTO mapper 세부 규칙의 source of truth가 아니다.

표준 문서와 충돌하면 표준 문서를 우선한다.

## 먼저 읽을 문서

현재 작업에 필요한 문서만 읽는다.

1. `${CLAUDE_PLUGIN_ROOT}/standards/tanstack-query-standard.md`
   - `queryOptions`, `mutationOptions`, query key factory, invalidation 규칙
   - `useSuspenseQuery`, `useQuery`, `mutate`, `mutateAsync` 정책
2. `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`
   - 레이어별 API / domain / query / mutation / hook 네이밍
   - generated SDK 또는 transport 이름 숨김
3. `${CLAUDE_PLUGIN_ROOT}/standards/test-driven-generation-workflow.md`
   - 사용자 영향 변경의 regression-first 기준
   - contract-only test와 behavior evidence 구분

## 작업별 라우팅

| 상황 | 시작점 |
| --- | --- |
| query/mutation 구현 또는 리뷰 | `tanstack-query-standard.md` |
| query key factory 또는 invalidation 설계 | `tanstack-query-standard.md` |
| domain mapper / DTO 경계 점검 | `dto-domain-boundary` 스킬 |
| 함수명, hook명, mutation 변수명 점검 | `function-naming.md` |
| 서버 상태 연결 전 regression test 설계 | `design-tests` 스킬 |

## 기본 경계

- external SDK와 generated type은 adapter/domain layer 뒤에 숨긴다.
- domain 또는 server-state layer는 SDK 호출, DTO alias, mapper, query/mutation contract, query key, mutation option, domain error normalization을 소유한다.
- feature는 UI orchestration, URL/search params, local interaction state, alert/toast, modal/drawer close, navigation, caller-specific side effect를 소유한다.
- feature와 page는 public query/mutation surface만 import한다.
- `api.ts`, `dto.ts`, `mappers.ts`, `errors.ts`, internal `rules.ts`는 feature import 대상이 아니다.
- 신규 서버 상태 파일은 현재 프로젝트의 domain/feature/shared 구조를 따른다.
- 승인되지 않은 generic bucket 파일을 만들지 않는다.

## 빠른 점검

TanStack Query 코드를 수정하기 전에 확인한다.

1. 대상 domain 또는 feature capability가 명시되어 있는가?
2. operation이 query, command, workflow 중 하나로 분류됐는가?
3. contract source가 명확한가: generated SDK/OpenAPI, REST sample, backend contract document, existing adapter.
4. feature/page가 public surface만 import하는가?
5. query key와 query function이 query factory 안에서 함께 정의되는가?
6. mutation 실행 상태가 caller에서 mutation object로 보존되는가?
7. cache invalidation이 문자열 literal이 아니라 key factory를 쓰는가?
8. regression 검증이 generated contract shape가 아니라 사용자 관찰 가능 동작을 보호하는가?

## Query Scaffold 기본값

새 서버 상태 surface를 만들 때는 편집 전 아래 shape를 먼저 제안한다.

```txt
domain-or-feature:
- dto: external contract aliases, if needed
- types: application-facing types
- mapper: DTO normalization and default/null policy
- api/adapter: transport call only
- query keys: cache ownership and invalidation targets
- queries: queryOptions or local equivalent
- mutations: mutationOptions or local equivalent
- hooks: useXxxQuery/useXxxMutation only when it adds a stable interface
- tests: behavior or contract coverage
```

프로젝트가 `domain/` 구조를 쓰지 않으면 강제하지 않는다. 기존 폴더 구조에 맞춰 같은 책임만 분리한다.

## Mutation 규칙

- mutation 이름은 HTTP method가 아니라 업무 command 의도를 드러낸다.
- caller에서 `useMutation()` 결과 객체를 유지한다.
- `mutate` alias를 action function처럼 빼지 않는다.
- UI side effect는 caller layer에 둔다.
- invalidation은 가능한 가장 좁은 query key를 사용한다.
- response result가 UI에 필요한 값이면 callback 호출 여부만 보지 말고 값을 검증한다.

## 피해야 할 것

- 문자열 query key를 컴포넌트에 흩뿌림
- domain boundary가 있는데 generated DTO type을 UI에 직접 import
- 가치 없이 `useQuery` 또는 `useSuspenseQuery`만 감추는 wrapper hook
- 좁은 key가 있는데 broad invalidation 사용
- HTTP method 이름을 public domain 이름으로 노출
- queryOptions/mutationOptions shape만 확인하는 low-value test

## 출력 형식

이 스킬을 사용한 뒤에는 아래만 보고한다.

```txt
TanStack Query 경계 점검
- 읽은 표준:
- 적용 surface:
- query/mutation 소유 파일:
- feature/caller 소유 파일:
- caller behavior 검증:
- 남은 리스크:
```

표준 전체를 반복하지 않는다. 현재 변경에 필요한 결정만 요약한다.
