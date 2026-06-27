# Component Composition Standard

> 화면 구성의 가독성을 위한 단일 원칙 기준.

- Status: Active v2.0
- Scope: 신규 및 수정되는 Sirloin OMS frontend 컴포넌트

## 1. 원칙

다음 사람이 부모 컴포넌트를 열었을 때, 화면의 데이터 흐름이 바로 보여야 한다.

이를 위해 두 가지만 지킨다:

- 부모 JSX는 서브컴포넌트의 선언적 조합이어야 한다.
- 기본은 분리하지 않는다. 명확한 기준에만 분리한다.

## 2. 분리 기준

두 결정을 구분한다. 둘 다 기본은 분리하지 않는다.

### 2.1 같은 파일 내에서 서브컴포넌트로 추출

부모 JSX에 조건부 표현(inline ternary, conditional prop 객체 조립, conditional handler 조립)이 2개 이상 흩어져 있을 때만 추출한다. 추출된 서브컴포넌트는 그 조건을 자체적으로 캡슐화한다.

### 2.2 별도 파일로 이동

다음 중 하나라도 성립할 때만 별도 파일로 옮긴다:

- 두 군데 이상에서 import되어 재사용된다.
- 파일 LOC가 `coding-style-guide` rule 11의 500줄 hard limit에 도달한다.
- 서브컴포넌트 단위로 자체 테스트를 작성해야 한다.

“책임이 분명해 보임”, “도메인 의미가 강함” 같은 주관적 이유로는 파일을 나누지 않는다.

## 3. 파일 내부 순서

상단부터 다음 순서로 배치한다.

1. imports
2. file-private types/interfaces
3. module-level 상수
4. 순수 helper 함수
5. exported 외부 컴포넌트 (entry)
6. orchestrator body 컴포넌트 (필요 시)
7. 서브컴포넌트 (render 순서대로)

## 4. 예시 (§2.1의 동일 원칙의 인스턴스)

아래 패턴은 모두 “조건부 표현을 서브컴포넌트가 캡슐화한다”는 같은 원칙의 적용 예시이다. 별도 규칙이 아니다.

```tsx
// inline ternary
<DetailErrorMessage message={errorMessage} />
// errorMessage falsy 시 내부에서 null 반환

// conditional prop 객체 조립
<DetailTable
  onLoadMore={onLoadMore}
  hasNextPage={hasNextPage}
  isFetchingNextPage={isFetchingNextPage}
/>
// 내부에서 infiniteScroll 객체 조립

// conditional handler 조립
<DetailExportFooter
  selection={selection}
  keyword={keyword}
  labels={labels}
  onExport={onExport}
/>
// 내부에서 handleExport와 request 조립
```

배열 children에 수동 key를 달아 넘기는 패턴은 fragment로 대체한다. narrowing용 ternary는 외부 컴포넌트에서 단 한 번만 허용된다.

## 5. Reference Exemplar

`src/features/stock/dateBucketStock/ui/DateBucketStockDetailDrawer.tsx`

- §2.1에 따라 7개 서브컴포넌트가 같은 파일 하단에 존재한다.
- §2.2 기준을 하나도 충족하지 않아 별도 파일로 나가지 않는다.
- §3 파일 내부 순서를 준수한다.

## 6. 기존 표준과의 관계

- `coding-style-guide` rule 11 (500줄 hard limit), rule 5 (`renderXxx()` 금지), rule 17 (조기 추상화 지양)과 정합한다.
- 도메인/feature 경계 결정은 `domain-feature-architecture-standard.md`의 구조를 따른다. 본 표준은 그 경계 안의 컴포넌트 구성만 다룬다.
- 기존 코드는 점진 적용, 신규 코드는 강제 적용.
