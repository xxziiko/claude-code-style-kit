# 자동화 경계 정의

> REST 표준화 작업에서 무엇을 자동화하고, 무엇을 사람이 결정해야 하는지 구분하기 위한 기준 문서

- Status: Draft
- Date: 2026-04-24
- Scope: Repo-wide

## 1. 목적

이 문서는 표준화 작업에서 자동화가 담당할 범위와 사람이 직접 판단해야 할 범위를 구분한다.

핵심 목표는 아래다.

1. 자동화가 표준을 강제하되, 비즈니스 설계를 대신하지 않게 한다.
2. scaffolding과 harness가 과도한 추론 없이 안전하게 동작하게 만든다.
3. "무엇을 생성할지"와 "무엇을 결정할지"를 분리한다.

## 2. 기본 원칙

자동화는 `결정적인 것`만 만든다.

사람은 `비결정적인 것`을 결정한다.

여기서 의미는 아래와 같다.

- 결정적인 것:
  - 입력이 같으면 항상 같은 결과가 나와야 하는 것
  - 표준 템플릿으로 안전하게 생성 가능한 것
  - 비즈니스 해석이 거의 필요 없는 것

- 비결정적인 것:
  - 도메인 경계 판단이 필요한 것
  - 비즈니스 의미 해석이 필요한 것
  - 여러 설계 대안이 가능한 것

## 3. 자동화 가능한 것

### A. Generated 계약 계층

- generated SDK / types 생성
- generated types에서 runtime contracts 추출
- `contracts.gen.ts` 생성

### B. 표준 파일 구조 생성

- `src/domain/{aggregate}/{capability}/`
- `api.ts`
- `types.ts`
- `queries.ts`
- `mutations.ts`
- concrete business rule files, for example `expirationDate.ts` or `supplierCode.ts`

### C. 표준 템플릿 삽입

- queryOptions factory 기본 골격
- mutationOptions factory 기본 골격
- value object smart constructor 기본 골격

### D. 로컬 검증

- forbidden import 검사
- direct `fetch` 검사
- `api.test.ts` 금지 검사
- default export / barrel export 검사
- codegen 후 contracts 생성 여부 검사

## 4. 사람이 결정해야 하는 것

### A. 도메인 경계

- 이 API가 어느 domain에 속하는가
- 이 기능이 query인지 command인지 workflow인지
- command를 별도 도메인으로 분리할지
- generated controller/path 이름이 업무 domain 이름과 다른지
- WMS generated SDK를 호출하더라도 실제 사용자가 수행하는 업무 aggregate가 `order`인지

### B. 비즈니스 의미

- generated union/enum-like 값이 실제로 value object가 되어야 하는가
- 어떤 값이 status인지, action인지, reason code인지
- 어떤 값이 UI에서 의미 있는가
- `commerce`가 자사몰 전용 정책인지, 단순 API 출처/외부 채널 이름인지
- 주문 command, 주문 배송 query, 출고/배송 제한(`orderDeliveryRestriction`) 중 어떤 capability인지

### C. mutation 후처리

- 어떤 query를 invalidate할지
- refetch가 필요한지
- alert / modal / navigation이 필요한지

### D. 테스트 시나리오

- 어떤 정책이 중요한가
- 어떤 실패가 문서화 가치가 있는가
- 어떤 orchestration 흐름을 보호해야 하는가

## 5. 반자동 영역

### A. domain scaffold

자동화:
- 파일 생성
- 표준 템플릿 채우기
- 승인된 generated SDK 연결
- 승인된 query/mutation/invalidation 패턴 연결

사람:
- domain 이름 확정
- query vs mutation 분리
- value object 필요 여부
- 실제 타입명과 비즈니스 의미 확정

### B. runtime contract 적용

자동화:
- generated union -> `contracts.gen.ts`

사람:
- 이 값을 실제 domain value로 쓸지 결정
- smart constructor 적용 여부 결정
- category / builder / rule 매핑 결정

### C. orchestration hook

자동화:
- 기본 hook 파일 템플릿

사람:
- 어떤 후처리를 넣을지
- 어떤 query를 갱신할지
- 어떤 UI 상태를 연결할지

## 6. 한 줄 기준

자동화가 답할 수 있는 질문:

- "이 파일 구조를 어떻게 만들지?"
- "이 표준 템플릿을 어디에 넣을지?"
- "generated 계약에서 어떤 런타임 값을 뽑을지?"
- "승인된 domain/mode 기준으로 `api.ts`, `types.ts`, `queries.ts`, `mutations.ts`를 어떻게 채울지?"

자동화가 답하면 안 되는 질문:

- "이 기능이 어느 도메인인가?"
- "이 값이 비즈니스적으로 어떤 의미인가?"
- "이 command 이후 어떤 화면 상태를 갱신해야 하는가?"

## 7. 권장 실행 순서

신규 REST API를 표준에 맞게 도입할 때의 권장 순서는 아래다.

1. `codegen:rest`
2. 에이전트가 generated SDK export와 `src/` import 스캔으로 미구현 candidate operation을 먼저 **보고**
3. 사용자가 대상 operation을 **선택**
4. 에이전트가 선택된 operation의 domain 모델링을 **검사**
5. 사용자가 그 제안을 **승인 또는 수정**
6. `scaffold:rest-domain`
7. 에이전트가 결정적인 구현을 **직접 채움**
8. caller migration이 필요하면 테스트 설계 단계로 handoff

핵심은:

- 에이전트는 판단을 **제안**할 수 있다.
- 하지만 그 제안은 domain 모델링 검사에 한정된다.
- 하지만 최종 도메인 경계 결정은 사용자가 승인해야 한다.
- 사용자가 표준 위반 방향을 제시하면 자동화는 이를 수행하지 않는다.
- scaffold와 그 이후의 결정적 구현은 승인된 입력만 사용한다.
- 테스트 작성과 caller migration은 별도 승인/검증 게이트를 통과해야 한다.

즉:

```txt
generated 계약 생성
  -> 미구현 candidate 보고
  -> 사용자 선택
  -> 에이전트 domain 모델링 검사
  -> 사용자 승인
  -> 표준 골격 생성
  -> 결정적 구현 자동 채움
  -> 테스트 설계/승인 게이트
  -> 회귀 보호막 통과 후 caller migration
```

## 8. 현재 보류 사항

아래는 이 문서에서 방향만 기록하고, 세부 스펙은 후속 논의로 남긴다.

1. domain 제안 workflow의 이름
2. 입력/출력 스키마
3. confidence / alternatives 표현 방식
4. 사용자 승인 UX

즉, 지금은 순서와 책임만 확정하고, workflow 세부 스펙은 나중에 구체화한다.
