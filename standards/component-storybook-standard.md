# 컴포넌트 / Storybook 표준

> 컴포넌트를 같은 역할 기준으로 일관되게 재사용하고 Storybook을 그 표준의 운영 기준으로 삼기 위한 문서

- Status: Baseline v1 (Agreed)
- Date: 2026-04-23
- Scope: Repo-wide

## 1. 목적

이 문서는 컴포넌트를 같은 역할 기준으로 일관되게 재사용하고, Storybook을 그 표준의 운영 기준으로 삼기 위한 문서다.

핵심 목표는 아래다.

1. 같은 역할의 UI가 화면마다 다른 방식으로 중복 구현되지 않게 한다.
2. shared / feature-specific / page-composed 구분을 명확히 한다.
3. Storybook을 문서가 아니라 컴포넌트 사용 기준의 source of truth로 만든다.

이 문서는 컴포넌트의 `분류 기준`과 Storybook 판단 기준을 다룬다.
디렉토리 구조의 최신 source of truth는 `docs/standards/shared-ui-conventions.md`와
`docs/standards/domain-feature-architecture-standard.md`를 따른다.

## 2. 컴포넌트 분류

### Shared component

- 여러 feature/page에서 재사용 가능
- 특정 도메인 문맥 없이도 의미가 유지됨
- 예: 입력, 선택, 테이블 셀, 공통 상태 표시, 공통 레이아웃 조각

신규 official shared UI 위치는 `src/shared/ui/{ComponentName}`이다.
기존 `src/components/**`는 리뷰 전까지 유지되는 legacy surface로 본다.

### Feature-specific component

- 특정 도메인/기능 문맥이 있어야 의미가 있음
- 여러 화면에서 재사용될 수 있지만 특정 feature에 묶임
- 예: 재고실사 detail panel, 출고 상태 action bar

위치는 기본적으로 해당 feature 디렉토리 아래

### Page-composed component

- 특정 페이지를 위해 조합된 화면 단위 컴포넌트
- 재사용보다 페이지 완성이 목적
- 다른 페이지에 복붙 재사용 대상이 아님

## 3. 현재 단계의 구조 원칙

지금 단계에서는 세부 폴더 구조보다 아래를 먼저 확정한다.

1. 어떤 컴포넌트가 `shared`인지
2. 어떤 컴포넌트가 `feature-specific`인지
3. 어떤 컴포넌트가 `page-composed`인지
4. 어떤 컴포넌트가 Storybook 대상인지

즉, 지금은 `구조를 확정`하는 단계가 아니라 `구조를 판단하는 기준을 확정`하는 단계다.

## 4. 상위 디렉토리 방향

신규 shared UI와 feature UI는 아래 target structure를 따른다.

```txt
src/shared/ui/{ComponentName}
src/features/{aggregate}/{capability}/ui
```

기존 `src/components/**` 파일은 일괄 이동하지 않는다. 변경 대상 컴포넌트를
리뷰할 때 public contract가 명확하고 official shared UI 기준을 만족하는 경우에만
승격하거나 이관한다.

## 5. 기본 원칙

1. 같은 역할의 UI가 이미 있으면 먼저 기존 컴포넌트를 검토한다.
2. 새 컴포넌트 생성보다 기존 표준 컴포넌트 확장/정리를 우선한다.
3. shared로 승격할 기준이 없으면 feature-specific에 둔다.
4. 페이지 전용 조합 컴포넌트를 성급하게 shared로 올리지 않는다.
5. 동작 중인 큰 컴포넌트의 내부 구조 정리는 후순위이며, 표준화에 직접 필요할 때만 한다.

## 6. 허용 / 금지 규칙

### 허용

- shared component를 여러 feature에서 재사용
- feature-specific component를 같은 feature 내 여러 화면에서 재사용
- Storybook을 기준으로 props/state 사용 예시를 문서화

### 금지

- 같은 역할의 새 UI를 별도 이름/스타일로 반복 생성
- shared component 없이 페이지마다 개별 구현을 계속 늘리는 것
- 비즈니스 로직이 강한 page-composed component를 성급하게 shared로 분리하는 것
- Storybook 대상 컴포넌트를 문서 없이 추가하는 것

## 7. 디렉토리 세부화 원칙

디렉토리 세부 설계는 아래 조건이 확보된 뒤 확정한다.

1. 같은 역할의 중복 컴포넌트 인벤토리가 작성되었는가
2. shared 승격 후보가 식별되었는가
3. Storybook 등록 우선순위가 정리되었는가
4. 실제 재사용 패턴이 한 번 이상 검증되었는가

따라서 지금은 세부 카테고리 이름이나 이동 규칙을 강하게 고정하지 않는다.

## 8. export 규칙

1. 일반 컴포넌트는 named export를 기본으로 한다.
2. 페이지 컴포넌트만 default export 예외를 허용한다.
3. barrel export는 금지한다.

## 9. 파일 크기 / 책임 규칙

1. 500줄 이상 파일은 분리 후보로 본다.
2. 다만 "크다"는 이유만으로 즉시 리팩토링하지 않는다.
3. 표준 적용이나 재사용 경계 정리에 직접 필요할 때만 우선 작업한다.

## 10. props / 상태 규칙

1. 컴포넌트는 자신의 관심사에 필요한 props만 받는다.
2. 비즈니스 로직은 가능하면 hook 또는 domain 경계로 분리한다.
3. JSX를 반환하는 `renderXxx()` 패턴보다 별도 컴포넌트 분리를 우선한다.
4. 상태는 local -> props -> context -> global 순으로 최소 범위를 선택한다.

## 11. 컴포넌트 표준화 절차

새로운 화면이나 수정 작업에서 아래 순서를 따른다.

1. 같은 역할의 기존 컴포넌트가 있는지 찾는다.
2. 있으면 그것을 재사용하거나 확장할 수 있는지 먼저 본다.
3. 없다면 feature-specific으로 먼저 만든다.
4. 재사용 패턴이 확인되면 shared 승격을 검토한다.
5. shared 또는 기준 feature component라면 Storybook에 등록한다.

## 12. Storybook 대상 우선순위

### 1순위

- `src/shared/ui/{ComponentName}`
- 여러 화면에서 반복 사용되는 official shared UI

### 2순위

- 여러 feature에서 재사용되는 조합형 컴포넌트
- 특정 feature 안에서 기준 역할을 하는 핵심 컴포넌트

### 3순위

- 복잡하지만 재사용 가치가 높은 feature-specific 컴포넌트

### 제외 또는 후순위

- 페이지 전용 조합 컴포넌트
- 라우팅/권한/페이지 데이터 의존이 강한 컴포넌트

## 13. Storybook 문서화 규칙

각 story는 가능하면 아래를 포함한다.

1. 기본 상태
2. 비어있는 상태
3. 로딩 또는 disabled 상태
4. 오류 또는 예외 상태
5. 상호작용이 있다면 대표 인터랙션 상태

## 14. Storybook의 역할

Storybook은 아래 역할을 가진다.

1. 기준 컴포넌트 식별
2. 상태별 사용 예시 기록
3. 신규 기여자가 어떤 컴포넌트를 써야 하는지 탐색하는 진입점
4. 컴포넌트 회귀 확인 보조 자산

## 15. Definition of Done

컴포넌트 표준화가 되었다고 보기 위한 최소 조건은 아래다.

1. 같은 역할의 기존 컴포넌트 존재 여부를 검토했다.
2. 새 컴포넌트가 필요한 이유가 설명 가능하다.
3. shared/component 분류가 타당하다.
4. export 규칙과 barrel 금지 규칙을 따른다.
5. Storybook 대상이면 story가 존재한다.
6. 신규 사용처가 기준 컴포넌트를 따르도록 정리되었다.
7. 디렉토리 구조 세부 결정이 아직 없더라도 분류 기준은 명확하게 설명 가능하다.
8. 사용자에게 보이는 동작/상태 변화가 있다면, 그 변화가 깨지지 않는 회귀 테스트가 통과했다.

회귀 테스트 기본 기준:

- 단순 표시 상태 변화 -> component test
- 사용자 상호작용 + 상태 전이 -> integration test
- 핵심 플로우 전반 변화 -> E2E 또는 시나리오 테스트

Storybook은 회귀 확인 보조 자산일 뿐이며, 회귀 테스트를 대체하지 않는다.

## 16. 후순위 리팩토링 원칙

아래는 기본적으로 후순위다.

- 이미 동작 중인 대형 컴포넌트의 내부 함수 분해
- 큰 파일을 읽기 좋게 나누는 정리성 리팩토링
- 동작/경계/재사용 표준에 직접 영향이 없는 가독성 개선

이 작업들은 표준 적용에 직접 필요할 때만 선행할 수 있다.

## 17. 후속 결정 항목

아래는 한 텀 적용 이후에 구체적으로 확정한다.

1. `shared` 하위 디렉토리 카테고리
2. feature-specific 컴포넌트의 세부 배치 규칙
3. page-composed 컴포넌트 분리 수준
4. 실제 폴더 이동 규칙

## 18. 현재 우선 과제

1. shared/component/page-composed 분류 기준 고정
2. 같은 역할의 중복 컴포넌트 사용처 인벤토리 작성
3. Storybook 대상 컴포넌트 우선순위 확정
4. 기준 컴포넌트 목록 작성
