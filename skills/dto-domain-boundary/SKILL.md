---
name: dto-domain-boundary
description: 외부 DTO와 Domain/UI 경계 규칙. DTO/Mapper/Domain 책임 분리와 적용 체크리스트. API 연동, 데이터 매핑, 리팩토링/구조개선 작업에서 우선 적용.
allowed-tools: Read, Grep, Glob, Edit, Write
---

# DTO-Domain Boundary Skill

외부 API 응답 타입, generated type, database row, serialized storage, third-party SDK object와
프론트엔드 Domain/UI 타입의 경계를 일관되게 유지하기 위한 가이드입니다.

> 외부 데이터를 화면 모델로 변환하는 작업에서는 이 스킬을 먼저 적용하세요.
> 네이밍과 서버 상태 규칙은 `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`,
> `${CLAUDE_PLUGIN_ROOT}/standards/tanstack-query-standard.md`를 함께 확인하세요.

---

## 목적

- API DTO와 Domain/UI 모델의 책임 분리
- null/default 정책의 단일화
- 훅 로직 단순화 및 재사용성 향상
- 스키마 변경 시 영향 범위 축소

---

## 용어 정의

- **DTO**: 외부 시스템에서 들어온 raw data shape. generated API type, HTTP response, DB row, SDK object를 포함한다.
- **Mapper**: DTO -> Domain/UI 타입 변환 함수
- **Domain Model**: 기능 규칙을 반영한 내부 타입
- **View Model**: 화면 렌더링에 최적화된 타입

---

## Critical Rules (MUST)

### 1) DTO는 외부 contract를 기준으로 정의

- 수동 Response interface를 새로 만들기보다 프로젝트의 generated type, schema type, typed fixture, backend contract를 우선 사용
- 필요한 경우 alias만 선언
- contract source가 불명확하면 추측으로 DTO를 만들지 않고 blocker로 보고

```typescript
type ProductListDto = ExternalProductListResponse["items"][number];
```

### 2) 변환 책임은 mapper에 집중

- 기본값(`??`), null 처리, 숫자 정규화, 파생값 계산은 mapper에서만 처리
- 훅/컴포넌트에 동일 정책을 분산하지 않음

### 3) Query hook은 조회 orchestration만 담당

- 쿼리 실행, loading/error, options 주입만 담당
- 도메인 규칙/검증 로직은 mapper 또는 domain factory로 분리

### 4) 에러 핸들링은 경계별 책임을 유지

- transport/client 에러는 adapter 또는 server-state layer에서 domain/user-facing error로 정규화
- UI side effect는 caller layer에서 처리
- 훅 내부 try/catch로 외부 클라이언트 에러를 의미 없이 재포장하지 않음

### 5) 외부 클라이언트 옵션은 adapter 경계에 둔다

- auth header, tenant/context, target API, base URL 같은 transport option은 adapter 또는 client boundary에서 일관되게 설정
- 컴포넌트가 transport option을 직접 알지 않게 한다.

---

## Recommended Rules (SHOULD)

### 1) 네이밍

- mapper 함수명: `mapXxxToYyy` 또는 `getXxxFromDto`
- DTO alias: `XxxDto` 접미사 사용 권장
- 화면 반환 타입: `XxxViewModel` 또는 기능명 타입

#### 네이밍 표준 (기본값)

| 대상 | 규칙 | 예시 |
|---|---|---|
| 외부 응답 항목 타입 | `XxxDto` | `ProductListItemDto` |
| 단일 변환 함수 | `mapXxxToYyy` | `mapProductDtoToProductSummary` |
| 배열 변환 함수 | `mapXxxListToYyyList` 또는 `mapXxxsToYyys` | `mapProductDtosToProductSummaries` |
| 화면 모델 타입 | `XxxViewModel` | `ProductSummaryViewModel` |
| 도메인 타입 | 도메인 의미 중심 명사 | `Customer`, `InventoryItem` |
| Smart constructor | `createXxx` | `createCustomerId`, `createEmailAddress` |

### 2) 파일 위치

프로젝트에 이미 정해진 구조가 있으면 그 구조를 우선한다. 명확한 기준이 없을 때의 기본값:

- feature-local UI state/hooks: `features/{feature}/hooks`
- feature-local view/display types: `features/{feature}/types.ts`
- DTO -> domain mapper: `domain/{domain}/{capability}/mappers.ts`
- domain query/command type: `domain/{domain}/{capability}/types.ts`
- API boundary params/payloads: `domain/{domain}/{capability}/types.ts`
- raw DTO aliases: `domain/{domain}/{capability}/dto.ts`
- smart constructor/business rule files: 구체적인 비즈니스 이름 사용, 예: `customerId.ts`, `dateRange.ts`

### 3) 승격 기준

- 2곳 이상에서 재사용되는 변환은 domain mapper로 이동
- 식별자 혼동 위험(code/id) 필드는 Branded Type 도입 검토
- 단순 field copy만 있는 mapper는 경계를 명확히 하는 목적이 있을 때만 둔다.

---

## 적용 절차

1. API 응답 경계 식별
2. DTO -> Domain/View mapper 작성
3. adapter/query function에서 mapper 호출만 수행
4. call site는 ViewModel/Domain 타입만 사용
5. 테스트 분리
   - mapper 단위 테스트
   - hook/component 통합 테스트

---

## 체크리스트

- [ ] 외부 응답 타입이 contract source 기반인가?
- [ ] DTO -> UI/Domain 변환이 mapper에만 존재하는가?
- [ ] 기본값/null 정책이 mapper 외부로 퍼지지 않았는가?
- [ ] 훅이 options 기반 extension point를 지원하는가?
- [ ] 훅 내부에 불필요한 try/catch가 없는가?
- [ ] 재사용 변환을 domain mapper로 승격해야 하는가?
- [ ] UI가 raw DTO/generated type을 직접 import하지 않는가?

---

## Testing Guardrails

- mapper는 **단위 테스트**로 기본값/null 처리/파생값 계산을 검증
- hook/component는 **통합 테스트**로 query 실행, options 주입, loading/error 상태를 검증
- 하나의 테스트에서 mapper 로직과 hook orchestration을 동시에 검증하지 않음
- 새로운 DTO 매핑 추가 시 복잡한 null/default/normalization 정책이 있으면 mapper 테스트를 함께 추가

---

## 예시 패턴

```typescript
// query hook
export function useProductSummariesQuery(options?: ProductSummariesQueryOptions) {
  return useQuery(productQueries.summaries(options));
}

// mapper
export const mapProductDtoToProductSummary = (
  dto: ProductDto,
): ProductSummary => ({
  id: createProductId(dto.id),
  name: dto.name ?? "",
  price: Number(dto.price ?? 0),
  isActive: dto.status === "ACTIVE",
});
```
