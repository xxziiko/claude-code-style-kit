---
name: coding-style-guide
description: 범용 TypeScript/React 프로젝트에서 코드 구조, 컴포넌트 조합, 훅 배치, TypeScript 스타일, 데이터 경계, 일반 리팩토링 규칙을 검토하거나 수정할 때 사용.
allowed-tools: Read, Grep, Glob, Edit, Write
---

# 코딩 스타일 가이드 라우터

이 스킬은 프로젝트를 일관된 코드 표준으로 이동시키는 라우팅 문서다.
세부 규칙의 source of truth가 아니며, 표준 문서와 충돌하면 표준 문서를 우선한다.

## 먼저 읽을 문서

현재 작업에 필요한 문서만 읽는다.

1. `${CLAUDE_PLUGIN_ROOT}/standards/typescript-code-style.md`
   - React import 규칙
   - `any` 금지
   - barrel/re-export 금지
   - 컴포넌트 export 규칙
2. `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`
   - 레이어별 함수, 훅, query/mutation factory 네이밍
   - generated SDK 또는 transport 이름 숨김
3. `${CLAUDE_PLUGIN_ROOT}/standards/tanstack-query-standard.md`
   - TanStack Query query/mutation factory, query key, invalidation 규칙
   - 서버 상태 훅과 UI 상태 훅 경계
4. `${CLAUDE_PLUGIN_ROOT}/standards/test-driven-generation-workflow.md`
   - 사용자 영향 변경의 regression/TDD 기준
   - 테스트 이름과 description 기준

## 작업별 라우팅

| 작업 | 읽을 문서 |
| --- | --- |
| React import, type-only import, explicit `any`, barrel export 점검 | `typescript-code-style.md` |
| 함수, 훅, query/mutation factory, handler 네이밍 | `function-naming.md` |
| DTO, mapper, API 응답, generated type, UI 데이터 경계 처리 | `function-naming.md`, `tanstack-query-standard.md` |
| TanStack Query query/mutation 구현 또는 리뷰 | `tanstack-query-standard.md` |
| 테스트 필요 여부, regression 보호막, 검증 근거 판단 | `test-driven-generation-workflow.md` |
| JSX 분리, 서브컴포넌트 추출, 조건부 렌더링 검토 | 이 스킬의 Compliance Checklist |
| shared hook / util / constant 생성 또는 승격 | 이 스킬의 현재 기본값 |

## 현재 기본값

- 신규 목표 구조는 현재 프로젝트의 기존 app/domain/feature/shared 경계를 우선한다.
- 명확한 local convention이 없으면 `domain`, `feature`, `shared` 같은 소유권 단위로 파일을 배치한다.
- feature에 결합된 훅은 feature-local `hooks`에 둔다.
- domain-agnostic hook / util / constant는 shared 영역에 둔다.
- domain 또는 transport 경계가 있는 프로젝트에서는 API 응답, DTO, generated type을 UI로 직접 새지 않게 한다.
- 서버 상태는 query/mutation factory, query key, invalidation 규칙을 한 위치에서 소유한다.
- UI는 orchestration, view state, form state, 표시 helper를 소유한다.
- `components/**`, `hooks/**`, `utils/**` 같은 generic bucket은 기존 구조가 아니라면 새 코드의 기본 위치로 쓰지 않는다.
- domain 개념을 별도 파일로 분리해야 하면 `model`, `read-model`, `query` 같은 추상 파일명이 아니라 구체적인 업무 이름을 사용한다.
- 승인된 구조가 없는 프로젝트에서 새 디렉터리 체계를 강제하지 않는다. 먼저 주변 패턴을 읽고 가장 작은 변경으로 맞춘다.

## 에이전트 판단 절차

에이전트는 코딩 표준 점검을 사람에게 넘기는 메모로만 작성하지 않는다.
아래 절차로 스스로 판정하고 다음 행동을 결정한다.

1. 변경 파일을 domain / feature / shared / legacy surface로 분류한다.
2. 변경 범위에 해당하는 표준 문서를 읽는다.
3. `Compliance Checklist`의 각 항목을 `pass`, `fail`, `n/a`로 판정한다.
4. `MUST fail`이 있으면 완료를 주장하지 않는다.
   - 즉시 수정 가능한 항목이면 수정한다.
   - 현재 범위에서 수정할 수 없으면 blocker로 보고한다.
5. `SHOULD fail`이 있으면 먼저 수정한다.
   - 유지가 더 적절하면 tradeoff와 예외 사유를 결과에 기록한다.
6. `MAY fail`은 현재 작업 범위를 넘으면 refactor queue 후보로 남긴다.
7. 최종 응답에는 판정 결과와 검증 명령을 함께 적는다.

## Compliance Checklist

> 이 체크리스트는 에이전트가 스스로 작업 완료 여부를 판정하기 위한 실행 기준이다.
> 사람이 나중에 리뷰할 수 있지만, 1차 목적은 에이전트 자기 판정이다.
>
> - **MUST**: 실패하면 완료를 주장하지 않는다. 수정하거나 blocker로 보고한다.
> - **SHOULD**: 실패하면 먼저 수정한다. 유지할 경우 tradeoff와 예외 사유를 기록한다.
> - **MAY**: 실패해도 현재 작업을 막지는 않는다. 필요하면 refactor queue 후보로 남긴다.

| Severity | 기준 | 에이전트 판정 기준 |
| --- | --- | --- |
| **MUST** | 소유 경계 | 신규/수정 파일이 현재 프로젝트의 app/domain/feature/shared 또는 이에 준하는 목표 구조 중 올바른 위치에 있는가? |
| **MUST** | DTO/generated 누출 금지 | UI, page, component, shared UI가 외부 DTO, generated type, raw API response를 직접 소비하지 않는가? |
| **MUST** | internal boundary import 금지 | 상위 UI 레이어가 하위 레이어의 internal `api`, `dto`, `mappers`, `errors`, `rules`를 직접 import하지 않는가? |
| **MUST** | 훅 배치 | feature-bound hook은 feature-local 위치에 있고, domain-agnostic hook은 shared 위치에 있는가? 신규 hook을 의미 없는 global bucket에 추가하지 않았는가? |
| **MUST** | TypeScript 안전성 | explicit `any`, 불필요한 barrel re-export, namespace `React.*` import를 새로 만들지 않았는가? |
| **MUST** | 컴포넌트 export | 일반 컴포넌트는 named export를 사용하고, framework routing 예외만 default export를 쓰는가? |
| **MUST** | 검증 | 사용자 영향 변경에 regression test, integration/E2E, story, manual verification 중 적절한 근거가 있는가? |
| **SHOULD** | 선언적 네이밍 | 함수/훅/handler/query/mutation 이름이 레이어별 function naming 표준을 따르고 generated/transport 이름을 그대로 새지 않게 숨기는가? |
| **SHOULD** | 컴포넌트 조합 | JSX 분리와 파일 분리가 복잡도/재사용/테스트 필요성에 근거하는가? |
| **SHOULD** | shared 승격 기준 | shared UI/hook/util 승격 시 domain import 금지, 좁은 public API, 재사용 근거를 만족하는가? |
| **SHOULD** | 응집도 | 재사용되지 않는 helper/type/config를 성급히 분리하지 않고, 파일 분리는 실제 복잡도에 근거하는가? |
| **SHOULD** | 부작용 가시성 | 함수 이름과 실제 side effect가 일치하고, mutation side effect나 cache invalidation이 숨겨지지 않는가? |
| **SHOULD** | 상수화 | magic number/string이 반복되거나 업무 의미를 가지면 명명된 상수로 추출했는가? |
| **MAY** | 파일 크기 | 파일이 과도하게 커져 탐색/리뷰 비용이 높아졌는가? 커졌다면 기준에 맞는 분리 후보를 남겼는가? |
| **MAY** | Props / Context | Props depth나 prop shape가 사용 관심사보다 넓어졌는가? 필요하면 composition/context 개선 후보를 남겼는가? |
| **MAY** | Form 응집도 | Form state가 field/form/feature hook 중 적절한 소유 위치에 있는가? |

### 에이전트 완료 판정

- `approved`: 모든 applicable MUST가 `pass`이고, 검증 명령이 통과했다.
- `blocked-by-must`: applicable MUST 중 하나 이상이 `fail`이며, 현재 범위에서 즉시 해결할 수 없다.
- `fixed-after-must-fail`: 처음에는 MUST가 실패했지만 에이전트가 수정했고 검증이 통과했다.
- `approved-with-should-notes`: MUST는 모두 통과했고 SHOULD 실패는 tradeoff/예외 사유가 기록됐다.
- `not-applicable`: 현재 변경이 이 스킬의 검토 범위가 아니다.

에이전트는 `blocked-by-must` 상태에서 완료 표현을 쓰지 않는다.
`SHOULD`와 `MAY`는 완료를 막지 않지만, 기록 없이 묵살하지 않는다.

## 출력 형식

이 스킬을 사용한 뒤에는 아래만 보고한다.

```txt
코딩 표준 점검
- 읽은 표준:
- 발견한 경계/스타일 이슈:
- 판정: approved | blocked-by-must | fixed-after-must-fail | approved-with-should-notes | not-applicable
- MUST:
- SHOULD:
- MAY:
- 검증:
```

표준 문서 전체를 반복하지 않는다. 현재 변경에 필요한 규칙만 적용한다.
