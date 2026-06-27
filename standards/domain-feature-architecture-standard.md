# Domain / Feature Architecture Standard

> Target architecture for DDD-oriented domain modeling and feature-sliced UI organization.

- Status: Draft v1
- Date: 2026-06-15
- Scope: New and changed Sirloin OMS frontend code

## 1. Purpose

This standard defines the target architecture for domain-based frontend code.

The goal is not to mirror backend entities or API response shapes. The goal is to
make the frontend's business language explicit and stable, so APIs with different
DTO names or shapes can be normalized into the same domain concepts when they
mean the same thing.

New and refactored code should make these boundaries clear:

```txt
api/generated -> domain -> features -> pages
```

Legacy surfaces still exist, but they are not the target architecture.

## 2. Core Intent

Domain types are not the average shape of API responses.

Domain types are the frontend's chosen business language:

- If multiple APIs return the same business concept using different names or
  shapes, mappers normalize those DTOs into the same domain concept.
- If multiple APIs share fields or IDs but represent different business
  meanings, do not merge them into a large shared type.
- Capability-specific projections are valid domain types when they represent a
  real business view.

Example:

```txt
product.stockItem
stock.stockItem
goodsItem.stock
inventoryItem
```

If these represent the same business concept, normalize them into a domain
concept such as `StockItem` or `StockItemRef`.

But do not merge these just because they share an ID:

```txt
StockOverviewItem
StockHistoryItem
StockExpirationItem
OrderLineStockItem
```

They may share `StockItemId`, but they can remain separate capability
projections when their business meaning differs.

## 3. Target Directory Shape

Use parallel domain and feature slices:

```txt
src/domain/{aggregate}/{capability}
src/features/{aggregate}/{capability}
```

Examples:

```txt
src/domain/stock/overview
src/features/stock/overview

src/domain/stock/history
src/features/stock/history

src/domain/stock/expiration
src/features/stock/expiration
```

Do not place feature UI directly under an aggregate:

```txt
src/features/stock/ui       # forbidden
src/features/stock/hooks    # forbidden
```

Every feature must have a capability/workflow boundary.

## 4. Domain Layer

Domain owns API boundaries, DTO normalization, frontend domain language,
query/mutation contracts, and business rules.

Standard shape:

```txt
src/domain/{aggregate}/
  branded.ts                # optional aggregate-wide branded primitive types

src/domain/{aggregate}/{capability}/
  index.ts                  # public domain capability surface
  api.ts                    # generated SDK/envelope/error boundary
  types.ts                  # capability query contract and api/query/mutation boundary types
  dto.ts                    # generated-adjacent DTO aliases, internal only
  mappers.ts                # DTO -> domain normalization
  queries.ts                # query keys and queryOptions
  mutations.ts              # mutationOptions
  errors.ts                 # optional domain error normalization
  rules.ts                  # optional domain policy/predicate helpers
  {businessName}.ts         # optional concrete business concept file
  __tests__/
```

Create only the files that are needed. Do not create empty standard files.

### `branded.ts`

`branded.ts` lives at the aggregate root when aggregate-wide branded primitive
types are useful.

```txt
src/domain/stock/branded.ts
src/domain/order/branded.ts
```

Use it for cross-capability identity boundaries, not for DTO aliases, UI types,
or business rules. Do not brand every string ID by default; brand IDs when
cross-use would create meaningful risk.

When a value object-like concept needs behavior, use an approved concrete
business filename such as `expirationDate.ts`, `stockQuantity.ts`, or
`dateRange.ts`.

### `types.ts`

`types.ts` contains capability-owned domain contract types:

- query result and business data contract
- api/query/mutation boundary params, payloads, and results
- IDs and branded types imported from aggregate `branded.ts`

Generated-adjacent raw response/request aliases belong in `dto.ts`, not
`types.ts`.

`types.ts` must not contain:

- generated DTO aliases
- table row types
- form value types
- drawer/modal state types
- display-only UI types

### Concrete Business Files

When a domain concept needs policy code, type guards, smart constructors, or
validation, use a concrete business filename rather than a generic bucket:

```txt
src/domain/stock/expirationDate.ts
src/domain/stock/stockQuantity.ts
src/domain/order/allocationResult.ts
```

Do not add generic bucket files outside the approved scaffold manifest.

### `dto.ts`

`dto.ts` is an internal generated-adjacent boundary.

Allowed:

```ts
import type { SomeGeneratedResponse } from '@/api/generated/wms';

export type StockExpirationResponseDto = SomeGeneratedResponse;
```

Forbidden:

```ts
export type { SomeGeneratedResponse } from '@/api/generated/wms';
```

DTO aliases are for `api.ts`, `mappers.ts`, and domain tests. They must not be
imported by `features/**`, `pages/**`, or `components/**`.

### `api.ts`

`api.ts` closes the generated/DTO/error/envelope boundary.

It owns:

- generated SDK calls
- request DTO construction
- response envelope handling
- domain error normalization
- mapper invocation
- returning domain types

It does not own DTO -> domain transformation logic directly. Put transformation
logic in `mappers.ts` and call it from `api.ts`.

External callers should receive domain types only.

### `queries.ts` and `mutations.ts`

Domain owns query and mutation contracts.

`queries.ts` owns:

- query keys
- queryOptions
- queryFn connection to `api.ts`
- cache boundary naming

`mutations.ts` owns:

- mutationOptions
- mutationFn connection to `api.ts`
- domain-level mutation contract

Feature code owns actual React Query execution and UX side effects.

### `errors.ts`

Use `errors.ts` when a capability needs to normalize generated, transport, or
backend error shapes into a stable domain error contract.

Domain errors may define variants and normalization helpers. They must not call
alert/toast, navigate, close modals, or encode feature-specific UX behavior.

### `rules.ts`

Rules are internal by default. Only export a rule through `index.ts` when a
feature genuinely needs a public pure domain predicate.

## 5. Domain Public Surface

Each domain capability has exactly one public surface:

```txt
src/domain/{aggregate}/{capability}/index.ts
```

Features, pages, and components may import from:

```ts
import {
  stockExpirationQueries,
  stockExpirationMutations,
  type StockExpirationItem,
} from '@domain/stock/expiration';
```

They must not deep import:

```ts
import { getStockExpirationList } from '@domain/stock/expiration/api';
import type { StockExpirationResponseDto } from '@domain/stock/expiration/dto';
import { mapStockExpirationDto } from '@domain/stock/expiration/mappers';
import { normalizeStockExpirationError } from '@domain/stock/expiration/errors';
import { stockExpirationQueries } from '@domain/stock/expiration/queries';
import type { StockExpirationItem } from '@domain/stock/expiration/types';
```

Internal by default:

- `api.ts`
- `dto.ts`
- `mappers.ts`
- `errors.ts`
- `rules.ts`

Public only through `index.ts`:

- domain types
- branded types when needed
- query factory
- mutation factory
- intentionally public domain predicates

## 6. Feature Layer

Features own user-facing workflows, UI, local interaction state, and feature
behavior.

Standard shape:

```txt
src/features/{aggregate}/{capability}/
  index.ts                  # public feature capability surface
  ui/
  hooks/
  types.ts
  config/
  utils/
  __tests__/
```

### `ui/`

React components for the feature capability.

### `hooks/`

Feature-specific React orchestration hooks.

Allowed:

- URL/search param orchestration
- local selection state
- drawer/modal interaction state
- feature flow hooks
- domain query/mutation usage composed with UX side effects

Forbidden:

- DTO mapping
- generated SDK imports
- domain business rule implementation
- table config-only files
- thin query wrappers that only hide `useSuspenseQuery(domainQueries.x())`

### `types.ts`

Feature-local UI/view/form/selection/display types only.

Feature types may compose domain types, but they must not redefine domain facts.

Examples:

```txt
StockExpirationTableRow
StockExpirationFormValues
StockExpirationColumnKey
StockExpirationSelectionState
```

### `config/`

Display and static UI configuration:

- columns
- tabs
- filter options
- drawer section config
- export column config

### `utils/`

Feature-local pure helpers. Do not place domain business rules here.

## 7. Feature Public Surface

Each feature capability has exactly one public surface:

```txt
src/features/{aggregate}/{capability}/index.ts
```

Pages may import from:

```ts
import { StockExpirationPageContent } from '@features/stock/expiration';
```

Pages must not deep import:

```ts
import { StockExpirationTable } from '@features/stock/expiration/ui/StockExpirationTable';
import { useStockExpirationSelection } from '@features/stock/expiration/hooks/useStockExpirationSelection';
```

Another feature should not depend on feature internals. If multiple features need
the same concept, first decide whether it belongs in domain or shared.

## 8. Pages

Pages are route shell and composition only.

Allowed:

- reading route params
- selecting layout boundaries
- rendering feature entry components
- placing Suspense/ErrorBoundary
- connecting route guards or permissions

Forbidden:

- generated SDK imports
- DTO imports
- domain query/mutation execution
- table/form/drawer implementation
- feature deep imports
- business rule implementation

## 9. Shared

`src/shared` is for domain-agnostic primitives.

Allowed:

- `shared/ui`: Button, DataTable, SideDrawer, SummaryTable
- `shared/hooks`: `useDebouncedValue`, `useTableSelection`, `useDisclosure`
- `shared/utils`: domain-agnostic date/string/number helpers
- `shared/constants`: app-wide constants

Forbidden:

- Stock/Order/Receipt/Supplier business language
- API DTOs
- domain mappers
- feature workflows
- screen-specific table columns
- domain-specific selection policy

Using something in two places is not enough reason to move it to shared.

## 10. Legacy / Transition Surfaces

These surfaces are legacy or transition surfaces:

```txt
src/components/**
src/hooks/**
src/interfaces/**
src/graphql/**
src/domain/**/ui
src/domain/**/hooks
```

Rules:

- Do not add new files to legacy surfaces.
- New features and refactors should use the target structure.
- Existing file edits are allowed when moving the code is too risky for the
  current task.
- If a legacy file is touched, report it as `legacy surface touched`.
- If migration is out of scope, add a refactor queue candidate.

## 11. Testing

Tests live beside the layer contract they protect.

Domain tests:

```txt
src/domain/{aggregate}/{capability}/__tests__/
```

Cover:

- mappers
- rules
- branded/domain helper behavior
- error normalization
- API boundary normalization when useful
- query/mutation factory contract when meaningful

Feature tests:

```txt
src/features/{aggregate}/{capability}/__tests__/
```

Cover:

- UI behavior
- feature hooks
- interaction flow
- mutation side effects
- regression behavior

## 12. Audit Policy

Apply this standard strictly to new and changed files.

Untouched legacy is report-only:

- New files must follow target architecture.
- Changed files must not make legacy boundaries worse.
- Large risky moves can be deferred, but the deferral must be explicit.
- Existing untouched violations are migration candidates, not blockers.

Audit summary should distinguish:

```txt
Blocking violation in changed code
Non-blocking legacy migration candidate
```
