# Create API Workflow Standard

> create-api workflow contract for generated API, domain scaffold, test planning, and caller migration.

- Status: Draft
- Date: 2026-04-24
- Scope: Repo-wide

## 1. 목적

이 문서는 create-api workflow이 어떤 입력을 받고,
어떤 순서로 동작하며,
어떤 출력과 승인 단계를 거쳐야 하는지 정의한다.

이 문서는 구현 전 운영 계약이다.

## 2. Workflow Definition

### 이름

- workflow 이름: `create-api`
- 사용자-facing entrypoints:
  - `/create-api`: lane/candidate selection
  - `/scaffold-api`: verified domain scaffold
  - `/connect-api`: feature/caller connection

### 역할

- 신규 REST API 도입의 lane/candidate router

### 책임

1. generated 계약 갱신
2. generated SDK 후보 확인
3. 선택된 operation의 request/response contract audit 수행
4. 선택된 operation에 대한 domain 모델링 검사 호출
5. 사용자 승인 지점 제시
6. `/scaffold-api`로 domain scaffold handoff
7. 필요 시 `/connect-api`로 caller/feature connection handoff

### 비책임

- 비즈니스 경계 최종 확정
- `/create-api` 단계의 도메인 구현 직접 완료
- 승인 없는 자동 생성
- 기존 GraphQL response shape를 REST contract로 승격

## 3. 입력 계약

### 필수 입력

- 대상 API 또는 generated operation
- 관련 서비스
  - `wms`
  - `oms`

### 선택 입력

- 후보 domain 이름
- 관련 endpoint/path
- 관련 request/response shape
- 사용자 메모

### 입력 예시

```txt
대상 operation: packingCancel
service: wms
endpoint: POST /api/v1/receipt/packing-cancel
```

## 4. 단계

## Phase 1. Contract Refresh

실행:

- `yarn codegen:rest`

성공 조건:

- generated SDK/types 갱신 성공
- runtime contracts 갱신 성공

실패 시:

- 즉시 중단
- proposal/scaffold 단계 진행 금지

## Phase 2. 후보 Operation 확인

실행:

- `src/`에서 `@api/generated/{service}/sdk.gen` import를 스캔
- generated SDK export 목록과 import 스캔 결과를 대조

출력:

- generated SDK에 있지만 `src/`에서 import되지 않은 candidate operation 목록
- 사용자 선택 필요 항목

규칙:

- `src/` import 스캔 없이 proposal로 넘어가지 않는다.
- 사용자가 대상 operation을 고르기 전에는 proposal로 넘어가지 않는다.
- 기존 GraphQL operation/type/fixture는 request/response shape의 source of truth가 아니다.

## Phase 2-1. Contract Audit

실행:

- 선택된 operation의 generated SDK function, endpoint, method, request body, response envelope/data path 확인
- generated 타입이 `unknown`이면 실제 dev/staging REST 샘플 또는 백엔드 REST 계약 문서 확인

출력:

- contract source: `generated SDK | OpenAPI | dev REST sample | backend contract doc`
- request body shape
- response envelope shape
- response data path
- success/error 판정 기준
- 기존 GraphQL wrapper와 다른 점

규칙:

- GraphQL variables wrapper나 GraphQL mutation field를 REST wrapper로 복사하지 않는다.
- `GraphQL wrapper 제거`는 REST 공통 envelope의 `data` 제거를 의미하지 않는다.
- mapper/test fixture는 contract audit의 REST shape를 따른다.

## Phase 3. Implementation Status 확인

실행:

- 선택된 operation이 generated SDK에 존재하는지 확인
- 선택된 operation이 이미 `src/`에서 `@api/generated/{service}/sdk.gen` import로 사용 중인지 확인

출력:

- selected operations generated 여부
- selected operations implemented 여부

규칙:

- 상태 JSON 파일이 아니라 실제 코드 import를 implementation 증거로 삼는다.
- 이미 구현된 operation을 새 후보로 다시 제안하지 않는다.
- 작업 중간 세션이 끊긴 경우에도 저장된 상태 파일 대신 다시 operation을 선택해 이어간다.

## Phase 4. Domain Modeling Inspection

실행:

- `rest-domain-proposal`

출력:

- API 의도 요약
- 추천 aggregate
- 추천 capability
- 추천 domain
- query/mutation 분류
- scaffold mode
- validation 전략
- 설계 계획
- standards decision
- scaffold 인자 초안
- 컨벤션 적합성 점검
- 근거
- confidence
- 대안

## Phase 5. Approval

사용자 승인 항목:

1. aggregate / capability
2. domain 이름
3. query/mutation 분류
4. scaffold mode
5. validation 전략
6. 설계 계획상 사람이 확정해야 하는 부분
7. scaffold 인자 초안

이 단계는 반드시 사람 승인을 받아야 한다.
승인 전에는 dry-run, scaffold, 파일 생성을 실행하지 않는다.

## Phase 6. Scaffold

실행:

- dry-run preview
- `yarn scaffold:rest-domain ...`

입력:

- 승인된 aggregate/capability
- 승인된 mode(내부 분류값)
- 승인된 query/mutation 정보
- `--service`

출력:

- 생성된 파일 목록
- 후속 deterministic implementation 대상 목록

규칙:

- 이 단계에서는 placeholder 테스트 파일을 생성하지 않는다.

## Phase 7. Deterministic Implementation

실행:

- `api.ts` generated SDK 연결
- `types.ts` domain surface 구현
- `queries.ts` / `mutations.ts` 표준 패턴 구현
- 승인된 invalidate key 반영

출력:

- 구현된 파일 목록
- 제거된 TODO 목록
- 남은 비결정적 TODO 목록

## Phase 8. Self Check

실행:

- `rest-api-conventions` self-check checklist

출력:

- pass / fail
- 위반 항목 목록

## Phase 9. Standards Check

실행:

- `standards:check`
- 필요 시 lint / typecheck

출력:

- 통과 / 실패
- 남은 위반 목록

## Phase 10. Test Workflow Handoff

출력:

- `design-tests` 필요 여부
- `connect-api` 필요 여부

규칙:

- 의미 있는 테스트는 이 단계 이후 `design-tests`로 먼저 설계한다.
- `write-tests`는 승인된 테스트 매트릭스가 있을 때만 실행한다.
- `connect-api`는 regression test 통과 증거가 있을 때만 실행한다.
- `scaffold-api`는 `write-tests` 또는 `connect-api`를 직접 실행하지 않는다.

## Phase 11. Handoff

출력:

- `Current stage`
- `Completed`
- `Validation`
- `Next agent actions`
- `Blocked on human`
- `Implementation status`

handoff 조건:

- `Next agent actions`는 자동 진행 허가가 아니다.
- 승인 게이트가 있으면 반드시 handoff한다.
- `Blocked on human`만 남았을 때만 handoff 가능

## 5. Critical Rules (MUST)

### MUST 1. codegen 변경 검토 선행

`codegen:rest` 직후에는 generated 변경점을 구조화해 읽어야 한다.

### MUST 1-1. REST contract source of truth 우선

REST adoption의 request/response source of truth 우선순위는 아래와 같다.

1. generated SDK/types와 OpenAPI contract
2. 실제 dev/staging REST 응답 샘플 또는 백엔드 REST 계약 문서
3. domain mapper/type/test fixture
4. 기존 GraphQL operation/type/fixture

GraphQL은 caller inventory와 기존 UI 후처리 확인용 참고자료다.
REST request body wrapper, response envelope, `data` 위치, success/error 판정 기준을
GraphQL shape에서 추정하면 안 된다.

### MUST 2. operation 선택 선행

proposal은 사용자가 고른 operation 기준으로만 수행한다.

### MUST 3. implementation scan 선행

`src/`의 `@api/generated/{service}/sdk.gen` import 스캔 기준으로 선택한 operation의 구현 여부를 확인해야 한다.

### MUST 3-1. contract audit 선행

deterministic implementation, test design, caller migration으로 넘어가기 전에 선택한 operation의
contract audit이 있어야 한다.

### MUST 4. 승인 전 scaffold 금지

사용자 승인 없이 scaffold를 실행하면 안 된다.

### MUST 5. generated 실패 시 중단

`codegen:rest`가 실패하면 이후 단계를 실행하지 않는다.

### MUST 6. proposal 출력은 구조화되어야 함

proposal은 자유서술이 아니라 정해진 형식으로 출력한다.

### MUST 7. proposal은 domain 모델링 검사만 수행해야 함

proposal은 실행 계획 자체를 평가하지 않는다.
선택된 operation의 domain 네이밍, mode, validation, 설계 계획, 컨벤션 적합성만 검사한다.

### MUST 8. 표준 위반 제안은 reject 해야 함

사용자 제안이 표준에 어긋나면 `reject`로 판정하고 scaffold를 진행하지 않는다.

### MUST 9. 승인된 deterministic implementation은 직접 채워야 함

표준과 승인으로 결정된 구현은 handoff TODO로 남기지 않는다.

### MUST 10. placeholder 테스트 생성 금지

scaffold-api 단계에서는 의미 없는 placeholder 테스트 파일을 만들지 않는다.

### MUST 11. scaffold는 승인된 입력만 받음

proposal 추천값을 임의로 재해석하지 않는다.

### MUST 12. self-check를 통과해야 함

결정적 구현 후 self-check를 통과하지 못하면 완료로 보고하지 않는다.

### MUST 13. standards check는 마지막에 반드시 실행

생성 후 검증 없이 완료로 보고하면 안 된다.

### MUST 14. 상태 기반으로 결과를 보고해야 함

결과는 TODO 나열이 아니라 현재 단계, 완료 범위, 검증 상태, 다음 에이전트 액션, 사람 결정 필요 항목으로 나누어 보고한다.

### MUST 15. implemented 판정은 검증 완료 이후만 가능

`Implementation status: implemented`는 domain 자산의 self-check와 standards:check가 통과한 뒤에만 가능하다.
caller migration 완료 또는 UI 통합 완료를 의미하지 않는다.
caller migration은 승인된 regression test 통과 결과가 별도로 있어야 진행한다.

## 6. 상태 기반 출력 계약

```txt
Current stage:
- <discovered|audited|designed|scaffolded|deterministic-implementation|verified|integrated|blocked>

Completed:
- generated candidate report: ...
- contract audit: ...
- implementation scan: ...
- scaffolded files: ...
- deterministic implementation: ...

Validation:
- domain modeling check: <pass|revise|reject>
- self-check: <pass|fail>
- standards:check: <pass|fail>
- regression test: <pass|fail|not-run>
- typecheck: <pass|fail|not-run>

Next agent actions:
- ...

Blocked on human:
- ...

Implementation status:
- selected operation: ...
- aggregate: ...
- capability: ...
- domain: ...
- status: <not-started|designed|scaffolded|implemented|deferred|rejected>
- notes: ...
```

## 7. Scaffold 입력 계약

workflow에서 scaffold에 넘기는 최소 인자:

```txt
--domain <name>
--mode <query|command|workflow>
--service <wms|oms>
--query <operation>         # optional
--mutation <operation>      # optional, repeatable
```

`mode`는 내부 분류값으로 사용하고, 사용자에게는 굳이 노출하지 않아도 된다.

## 8. 실패 처리

### generated 실패

- 흐름 중단
- 에러 출력
- 사용자에게 해결 필요 사항 전달

### proposal 불명확

- confidence가 낮으면 대안과 함께 재질문
- 승인 없이 다음 단계 진행 금지

### scaffold 실패

- 생성된 파일 목록과 실패 지점 출력
- 부분 생성물이 있으면 명시

### standards check 실패

- 완료로 보고하지 않음
- 위반 항목을 구조화해 출력

## 9. 완료 판정

`/scaffold-api` 흐름 완료는 아래를 모두 만족할 때만 인정한다.

1. generated 갱신 성공
2. candidate operation 요약 완료
3. implementation scan 완료
4. 사용자가 대상 operation 선택 완료
5. proposal 출력 완료
6. 사용자 승인 완료
7. scaffold 성공
8. deterministic implementation 완료
9. self-check 통과
10. standards check 성공
11. 필요 시 테스트 워크플로우 handoff 완료
12. caller regression은 별도 `/connect-api` 단계로 위임
13. 상태 기반 보고 제공

## 10. 한 줄 정의

`/create-api`은 후보 선택 라우터이고, `/scaffold-api`는 승인 가능한 반자동 REST domain scaffold 실행기다.
