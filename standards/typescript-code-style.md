# TypeScript 코드 스타일

Status: Active
Updated: 2026-05-20

## React import 규칙

### `import React` 네임스페이스 사용 금지

`React.ReactNode`, `React.ReactElement`, `React.ComponentProps`, `React.Children`, `React.isValidElement` 등
`React.~~` 형태의 네임스페이스 접근을 사용하지 않는다.

필요한 타입과 API는 `'react'`에서 직접 named import한다.

**금지**

```ts
import React from 'react';

// 타입
children: React.ReactNode;
function render(ui: React.ReactElement) { ... }
type Props = React.ComponentProps<typeof Foo>;

// 런타임 API
const items = React.Children.toArray(children);
if (React.isValidElement(child)) { ... }
```

**허용**

```ts
import { type ReactNode, type ReactElement, type ComponentProps, Children, isValidElement } from 'react';

// 타입
children: ReactNode;
function render(ui: ReactElement) { ... }
type Props = ComponentProps<typeof Foo>;

// 런타임 API
const items = Children.toArray(children);
if (isValidElement(child)) { ... }
```

### 타입 전용 import는 `type` 키워드 사용

런타임에 값으로 필요하지 않은 import는 `type` 키워드를 붙여 타입 전용 import임을 명시한다.
이렇게 하면 번들러가 타입 import를 완전히 제거할 수 있다.

**금지**

```ts
import { ReactNode, ComponentProps } from 'react';
```

**허용**

```ts
import { type ReactNode, type ComponentProps } from 'react';
// 또는 런타임 값과 함께
import { Children, isValidElement, type ReactNode } from 'react';
```

### 예외: JSX 자동 변환

React 17+ 의 automatic JSX runtime을 사용하므로 JSX 사용만을 위한 `import React` 는 불필요하다.
JSX를 사용하는 파일에서 별도로 React를 import하지 않아도 된다.

## 타입 선언 규칙

### 명시적 `any` 사용 금지

`any` 타입을 명시적으로 사용하지 않는다. ESLint `@typescript-eslint/no-explicit-any` 규칙으로 경고한다.

대신 다음을 사용한다.

- `unknown` — 타입을 알 수 없는 외부 데이터
- 제네릭 — 호출 시점에 타입을 결정해야 하는 경우
- 구체적인 union 타입 — 가능한 값이 제한적인 경우

## Re-export 패턴 금지

barrel export(`index.ts`를 통한 re-export)를 만들지 않는다.
항상 원본 모듈에서 직접 import한다.

**금지**

```ts
// shared/ui/index.ts
export { SideDrawer } from './SideDrawer/SideDrawer';
export { SearchField } from './SearchField/SearchField';
```

**허용**

```ts
// 직접 import
import { SideDrawer } from '@shared/ui/SideDrawer/SideDrawer';
import { SearchField } from '@shared/ui/SearchField/SearchField';
```
