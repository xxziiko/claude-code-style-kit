# QA E2E Coverage Harness

> Playwright E2E 실행 결과, TC 단위 자동화 상태, 브라우저 실행 코드 커버리지를 함께 수치화하기 위한 하네스 설계

- Status: Draft
- Date: 2026-05-27
- Scope: Playwright E2E, QA TC manifest, E2E coverage reporting

## 1. 목적

이 문서는 QA TC를 기준으로 E2E 테스트 작성과 커버리지 관리를 수치화하는 방법을 정의한다.

핵심 목표:

1. AI가 작성한 E2E 테스트의 결과를 사람이 검토 가능한 숫자로 만든다.
2. 라인 커버리지와 TC 자동화율을 분리해 과신 지표를 막는다.
3. Playwright 실행 결과, TC manifest, screenshot evidence, Istanbul coverage를 하나의 리포트로 모은다.
4. 한 PR에서 측정 하네스와 pilot feature 수치화를 함께 도입할 수 있는 최소 범위를 정의한다.

## 2. 기본 원칙

### TC가 상위 source of truth다

E2E line coverage는 브라우저가 실제로 실행한 FE 코드의 양을 보여주는 보조 지표다.
하지만 line coverage는 화면 mount, layout, 공통 hook, table, shared component 실행으로 쉽게 부풀 수 있다.

따라서 QA 품질의 주 지표는 line coverage가 아니라 TC 단위 자동화 상태다.

### Coverage는 세 층으로 나눈다

1. TC Coverage: 사람이 만든 시나리오가 얼마나 자동화됐는가
2. Run Quality: 자동화된 테스트가 얼마나 안정적으로 통과하는가
3. Code Execution Coverage: E2E가 실제 FE 코드를 얼마나 지나갔는가

### PR gate는 coverage percent만으로 판단하지 않는다

`line coverage >= N%`는 단독 gate가 될 수 없다.
PR gate는 protected TC가 실행됐는지, PASS/FAIL이 명확한지, reviewer가 확인할 evidence가 있는지를 함께 본다.

## 3. 지표 정의

각 레이어는 서로 다른 질문에 답한다.
하나의 숫자로 전체 품질을 대표하지 않고, 아래 지표들을 함께 읽는다.

| Layer                  | Tool / Source              | 검증하려는 것                                         | 수치화 지표                                   |
| ---------------------- | -------------------------- | ----------------------------------------------------- | --------------------------------------------- |
| Test case source       | QA TC manifest             | 사람이 만든 시나리오가 자동화 대상으로 관리되는가     | TC automation rate, total resolved rate       |
| Static React quality   | React Doctor               | AI가 만든 React 코드에 정적 품질 문제가 새로 생겼는가 | React Doctor score, new high findings         |
| Existing static checks | ESLint, TypeScript         | 팀 규칙과 타입 계약을 지키는가                        | lint pass/fail, typecheck pass/fail           |
| Runtime behavior       | Playwright result          | 사용자가 보는 시나리오가 실제 브라우저에서 통과하는가 | E2E pass rate, fail count, skip count         |
| Review evidence        | Screenshot / trace / video | PASS가 정말 의도한 화면을 검증했는가                  | evidence rate, missing evidence count         |
| Code execution         | Istanbul coverage          | E2E가 FE 코드 어디를 실제로 실행했는가                | line, branch, function, changed-file coverage |

### TC 지표

| Metric              | Formula                               | Meaning                                    |
| ------------------- | ------------------------------------- | ------------------------------------------ |
| TC total            | all rows                              | manifest에 등록된 전체 TC 수               |
| Automatable         | `feasibility = automatable`           | FE E2E로 검증 가능한 TC 수                 |
| Automated           | `automation_status = automated`       | spec에 연결된 TC 수                        |
| Manual              | `automation_status = manual`          | 수동 검증으로 남긴 TC 수                   |
| Spec-out            | `automation_status = spec_out`        | 최신 스펙에서 제외된 TC 수                 |
| Todo                | automatable - automated               | 자동화 가능한데 아직 자동화하지 않은 TC 수 |
| TC automation rate  | automated / automatable               | 자동화 가능 TC 중 자동화된 비율            |
| Total resolved rate | automated + manual + spec_out / total | 전체 TC 중 처리 상태가 결정된 비율         |

### Run 지표

| Metric        | Formula                              | Meaning                                    |
| ------------- | ------------------------------------ | ------------------------------------------ |
| E2E pass rate | passed / executed                    | 실행된 자동화 TC 중 통과 비율              |
| Fail count    | failed                               | 실패한 자동화 TC 수                        |
| Skip count    | skipped                              | 실행되지 않은 자동화 TC 수                 |
| Evidence rate | evidence_attached / passed           | PASS TC 중 screenshot evidence가 있는 비율 |
| Flaky signal  | retry_passed or inconsistent results | 재시도 후 통과 또는 실행 간 결과 흔들림    |

### Code coverage 지표

| Metric                | Source                       | Meaning                             |
| --------------------- | ---------------------------- | ----------------------------------- |
| Line coverage         | Istanbul coverage summary    | E2E 중 실행된 line 비율             |
| Branch coverage       | Istanbul coverage summary    | E2E 중 실행된 branch 비율           |
| Function coverage     | Istanbul coverage summary    | E2E 중 실행된 function 비율         |
| Changed-file coverage | coverage map + changed files | 변경 파일 중 E2E가 실행한 파일 비율 |

### React Doctor 지표

| Metric             | Source                | Meaning                                   |
| ------------------ | --------------------- | ----------------------------------------- |
| React Doctor score | `react-doctor` report | 변경 후 React 정적 품질 점수              |
| New findings       | diff scan             | 이번 변경에서 새로 생긴 진단 수           |
| New high findings  | diff scan             | 이번 변경에서 새로 생긴 high 이상 진단 수 |
| Scanned files      | diff scan             | React Doctor가 검사한 변경 파일 수        |

React Doctor는 ESLint나 TypeScript를 대체하지 않는다.
AI-generated React code에서 state/effect, performance, architecture, accessibility, security 계열의 정적 품질 문제를 보조로 잡는 레이어다.

### 레이어별 보장 범위와 활용 방식

이 하네스는 “테스트가 존재한다”를 보장하는 도구가 아니라, AI가 작성한 테스트가 사람이 만든 TC와 실제 실행 결과, evidence, code execution, 정적 품질 신호에 연결되어 있는지를 확인하는 도구다.
각 레이어는 서로 다른 실패 모드를 잡기 때문에 하나의 지표로 합치지 않고 함께 읽는다.

#### TC source 레이어

보장할 수 있는 것:

1. QA TC 중 어떤 행이 FE E2E 자동화 대상인지 구분한다.
2. 어떤 TC가 실제 Playwright spec과 연결됐는지 추적한다.
3. 자동화 가능하지만 아직 작성하지 않은 TC를 `todo`로 드러낸다.
4. `manual`, `spec_out`, `blocked`, `needs_review`를 자동화 실패와 섞지 않고 따로 관리한다.

활용 상황:

1. PR 리뷰에서 “이번 변경이 어떤 QA TC를 자동화했는가”를 확인한다.
2. AI가 작성한 spec이 실제 어떤 TC를 대표하는지 검수한다.
3. QA/FE/PM이 자동화 불가능한 TC와 스펙아웃 TC를 같은 표에서 정리한다.

얻는 성과:

1. line coverage 대신 사람이 이해할 수 있는 `TC automation rate`를 주 지표로 둔다.
2. AI가 작성한 테스트를 “그럴듯한 코드”가 아니라 “특정 TC를 검증하는 코드”로 리뷰한다.

필요한 이유:

AI는 자연어 시나리오를 그럴듯한 Playwright 코드로 바꿀 수 있지만, 그 코드가 실제로 어떤 TC를 대표하는지 스스로 안정적으로 증명하지 못한다.
TC source 레이어가 없으면 테스트 개수, spec 파일 수, line coverage 같은 대체 지표에 기대게 되고, QA가 기대한 사용자 시나리오가 빠져도 놓치기 쉽다.

#### Runtime 레이어

보장할 수 있는 것:

1. 자동화된 TC가 실제 브라우저에서 `passed`, `failed`, `skipped`, `unmapped` 중 어떤 상태인지 계산한다.
2. manifest에는 `automated`로 등록됐지만 Playwright 결과와 연결되지 않은 TC를 찾는다.
3. spec title 또는 file path 변경으로 TC 연결이 깨진 경우를 드러낸다.

활용 상황:

1. P0 TC가 실제 브라우저에서 통과했는지 PR gate 또는 warning으로 확인한다.
2. E2E 실패를 개발자 test title이 아니라 QA TC 단위로 보고한다.
3. AI가 spec 파일만 만들고 실행 가능한 상태까지 책임지지 못한 경우를 잡는다.

얻는 성과:

1. 결과 표현이 “spec 2개 passed”에서 “TC 2개 passed”로 바뀐다.
2. QA/기획/개발이 같은 TC 단위로 실패와 완료 상태를 이야기할 수 있다.

필요한 이유:

AI가 만든 테스트 코드는 작성 시점에는 맞아 보여도, 실제 dev server, auth setup, network mock, browser assertion까지 통과해야 의미가 있다.
Runtime 레이어는 테스트 코드의 존재와 실제 사용자 흐름 검증 사이의 간극을 줄인다.

#### Evidence 레이어

보장할 수 있는 것:

1. `evidence_required=true`인 TC가 PASS했을 때 screenshot evidence가 남았는지 확인한다.
2. PASS 결과가 실제로 사람이 의도한 화면을 검증했는지 빠르게 검토할 수 있는 PNG를 남긴다.
3. `toBeHidden`, redirect, loading fallback처럼 false positive가 생기기 쉬운 테스트를 사람이 확인할 수 있게 한다.

활용 상황:

1. AI가 작성한 E2E PR에서 reviewer가 PASS TC의 화면만 빠르게 훑는다.
2. QA가 전체 수동 재검수를 하지 않고도 핵심 화면 evidence를 확인한다.
3. selector는 맞지만 검증 강도가 약한 테스트를 찾아낸다.

얻는 성과:

1. 코드 리뷰만으로 놓치기 쉬운 무의미한 PASS를 줄인다.
2. 수동 QA 비용을 “전체 재실행”이 아니라 “PASS evidence 검토”로 낮춘다.

필요한 이유:

E2E 테스트의 PASS는 “assertion이 만족됐다”는 뜻이지 “사람이 기대한 화면이 맞다”는 뜻은 아니다.
AI가 만든 테스트일수록 selector가 우연히 맞거나, UI가 전부 깨졌는데 hidden assertion만 통과하는 식의 false positive가 생길 수 있다.
Evidence 레이어는 이 마지막 의미 검증을 사람에게 넘기기 위한 안전장치다.

#### Code execution coverage 레이어

보장할 수 있는 것:

1. E2E 중 실제 실행된 FE file, line, branch, function 비율을 확인한다.
2. 대상 페이지 진입 시 어떤 layout, hook, shared component, route 코드가 실행됐는지 탐색한다.
3. coverage HTML로 실행된 코드와 실행되지 않은 코드를 확인한다.

보장하지 않는 것:

1. line coverage가 높다고 TC 품질이 높다는 뜻은 아니다.
2. 한 페이지 mount만으로도 공통 코드가 많이 실행되어 수치가 부풀 수 있다.
3. 따라서 code coverage는 주 지표가 아니라 보조 진단 지표다.

활용 상황:

1. E2E가 실제 대상 화면 코드까지 도달했는지 확인한다.
2. 변경 파일이 있는데 E2E 실행 coverage가 0인 경우 warning 후보로 삼는다.
3. 추후 changed-file coverage를 붙여 PR 영향 범위와 E2E 실행 경로를 비교한다.

얻는 성과:

1. E2E가 빈 껍데기인지, 실제 FE 실행 경로를 탔는지 진단한다.
2. TC coverage와 함께 읽어 “시나리오는 통과했지만 실행 범위는 낮다” 또는 “실행 범위는 높지만 TC는 적다” 같은 해석이 가능해진다.

필요한 이유:

TC 단위 지표만 있으면 테스트가 어떤 코드 경로를 실제로 지나갔는지 알기 어렵고, line coverage만 있으면 어떤 사용자 시나리오를 검증했는지 알기 어렵다.
두 지표를 분리해서 함께 두면 E2E 품질을 과신하지 않으면서도 실행 경로 진단에 필요한 데이터를 얻을 수 있다.

#### React Doctor 정적 품질 레이어

보장할 수 있는 것:

1. React Doctor 실행 결과 또는 skipped 사유가 QA E2E report에 남는다.
2. AI-generated React code에서 state/effect, performance, architecture, accessibility, security 계열의 정적 품질 신호를 별도 레이어로 추적할 수 있다.
3. ESLint/TypeScript와 별개로 React 코드 품질 warning gate로 확장할 수 있는 자리를 만든다.

현재 보장하지 않는 것:

1. 기본 실행에서는 `RUN_REACT_DOCTOR=1`이 없으면 scan하지 않고 skipped report만 남긴다.
2. React Doctor finding은 아직 PR을 block하지 않는다.
3. score 또는 finding schema는 프로젝트 baseline과 false positive를 확인한 뒤 gate로 승격한다.

활용 상황:

1. AI가 E2E뿐 아니라 React 화면 코드도 수정한 PR에서 advisory scan을 실행한다.
2. 기존 ESLint가 잡지 못하는 state/effect, architecture, accessibility 계열의 신호를 reviewer에게 제공한다.
3. baseline이 쌓인 뒤 `new high findings` 또는 `new critical findings`를 advisory policy로 확장한다.

얻는 성과:

1. “AI가 만든 테스트”와 “AI가 만든 React 코드”를 같은 QA report에서 함께 볼 수 있다.
2. 테스트가 통과하더라도 React 코드 품질 리스크가 새로 생겼는지 별도로 추적한다.

필요한 이유:

E2E가 통과해도 React 코드의 effect dependency, unnecessary render, accessibility, state ownership 문제는 남을 수 있다.
특히 AI-generated React code는 동작은 맞지만 구조적으로 약한 코드가 생기기 쉬우므로, Runtime/Evidence 레이어와 별개로 정적 품질 레이어가 필요하다.

#### 종합 성과

현재 하네스가 최종적으로 말할 수 있는 문장은 아래와 같다.

```txt
이 TC는 자동화 대상이고,
이 Playwright spec과 연결되어 있으며,
실제 브라우저에서 통과했고,
사람이 볼 evidence가 남았고,
실행 중 지나간 FE coverage는 이 정도이며,
AI-generated React code에 대한 정적 품질 신호는 이 상태다.
```

즉, 이 구조는 완전 자동 QA가 아니라 에이전트에게 테스트 작성과 실행을 위임하면서도 임의 확정하면 안 되는 판단 지점을 숫자와 산출물로 고정하는 하네스다.
에이전트가 맡는 영역은 spec 작성, 실행, 결과 매핑, evidence 생성, 보조 진단 수집이다. TC 최신성, FE bug와 TC mismatch의 구분, PASS 화면 의미 검토, React Doctor finding의 실제 처리 여부는 근거가 부족하면 `blocked-on-decision` 또는 `advisory-review`로 분리한다.

## 4. 파일 구조

```txt
docs/
  qa/
    e2e-tc-manifest.csv
scripts/
  qa-e2e/
    clean-e2e-coverage.mjs
    merge-e2e-coverage.mjs
    generate-qa-e2e-report.mjs
    run-react-doctor.mjs
e2e/
  fixtures/
    coverage-test.ts
.nyc_output/                # generated, gitignored
coverage-e2e/               # generated, gitignored
.omx/
  qa/
    runs/
      <run-id>/
        playwright-results.json
        react-doctor-report.json
        tc-results.json
        coverage-summary.json
        report.md
        evidence/
```

`docs/qa/e2e-tc-manifest.csv`는 repo에 남기는 source of truth다.
`.omx/qa/runs/**`, `.nyc_output/**`, `coverage-e2e/**`는 실행 산출물이다.

## 5. TC Manifest

최소 schema:

```csv
tc_id,feature,priority,title,feasibility,automation_status,spec_path,test_title,evidence_required,notes
ORD-001,order-cancel,P0,주문 취소 성공,automatable,automated,e2e/order/order-cancel.spec.ts,주문 취소 성공 후 목록에서 제거된다,true,
ORD-002,order-cancel,P1,외부 결제 취소 확인,manual,manual,,,false,외부 결제 의존
ORD-003,order-cancel,P2,삭제된 기능 확인,spec_out,spec_out,,,false,기획 변경
```

### `feasibility`

| Value          | Meaning                                                 |
| -------------- | ------------------------------------------------------- |
| `automatable`  | FE E2E에서 검증 가능                                    |
| `manual`       | POS, 실제 결제, 배치, 외부 시스템 등 수동 검증 필요     |
| `spec_out`     | 최신 스펙에서 제외                                      |
| `blocked`      | 자동화 가능성은 있으나 테스트 API, 계정, 환경 등이 막힘 |
| `needs_review` | LLM 또는 작성자가 판단했지만 명시적 의사결정 필요       |

### `automation_status`

| Value       | Meaning                          |
| ----------- | -------------------------------- |
| `automated` | Playwright spec에 연결됨         |
| `todo`      | 자동화 가능하지만 아직 spec 없음 |
| `manual`    | 수동 검증 대상으로 확정          |
| `spec_out`  | 스펙아웃으로 확정                |
| `archived`  | 더 이상 관리하지 않는 과거 TC    |

## 6. Playwright Result Mapping

초기 MVP에서는 TC와 Playwright test를 아래 조합으로 매핑한다.

```txt
spec_path + test_title
```

이 방식은 기존 test title을 크게 바꾸지 않고 manifest만으로 연결할 수 있다.
다만 title 변경에 취약하므로, 하네스가 안정화되면 Playwright annotation 또는 title prefix로 `tc_id`를 직접 심는 방식을 검토한다.

예:

```ts
test('주문 취소 성공 후 목록에서 제거된다', async ({ page }) => {
  // ...
});
```

Manifest:

```csv
tc_id,spec_path,test_title
ORD-001,e2e/order/order-cancel.spec.ts,주문 취소 성공 후 목록에서 제거된다
```

## 7. E2E Code Coverage

아티클의 Playwright + Istanbul 방식은 code execution coverage 수집 레이어로 사용한다.

흐름:

```txt
E2E_COVERAGE=1 yarn test:e2e
  -> Vite instrumentation
  -> window.__coverage__
  -> .nyc_output/*.json
  -> merge-e2e-coverage.mjs
  -> coverage-e2e/index.html
  -> coverage-e2e/coverage-summary.json
```

### Vite instrumentation

`E2E_COVERAGE=1`일 때만 `src/**/*.{ts,tsx,js,jsx}`에 Istanbul instrumentation을 적용한다.
일반 개발 서버와 기본 E2E 실행에는 영향을 주지 않아야 한다.

coverage 전용 Playwright config는 `http://localhost:3101`에서 별도 dev server를 띄운다.
기존 `dev:msw` 서버를 재사용하면 비계측 bundle이 실행되어 `window.__coverage__`가 비는 문제가 생길 수 있기 때문이다.

### Playwright fixture

coverage mode에서는 각 테스트 종료 후 `window.__coverage__`를 읽어 `.nyc_output/*.json`에 저장한다.
기존 모든 spec을 한 번에 바꾸지 않고, pilot feature spec부터 coverage fixture를 적용한다.

## 8. Report

`generate-qa-e2e-report.mjs`는 아래 입력을 읽는다.

1. `docs/qa/e2e-tc-manifest.csv`
2. Playwright JSON reporter output
3. `coverage-e2e/coverage-summary.json`
4. React Doctor JSON/report output
5. evidence directory

최소 출력:

```txt
.omx/qa/runs/<run-id>/tc-results.json
.omx/qa/runs/<run-id>/report.md
```

Report 예시:

```md
# QA E2E Report

## Summary

- TC total: 32
- Automatable: 27
- Automated: 21
- Manual: 3
- Spec-out: 2
- Todo: 6
- Passed: 20
- Failed: 1
- TC automation rate: 77.8%
- E2E pass rate: 95.2%
- Evidence rate: 90.0%
- Line coverage: 61.2%
- React Doctor score: 88
- React Doctor new high findings: 0

## By Feature

| Feature      |  TC | Auto | Manual | SpecOut | Todo | PASS | FAIL | Evidence |
| ------------ | --: | ---: | -----: | ------: | ---: | ---: | ---: | -------: |
| order-cancel |  32 |   21 |      3 |       2 |    6 |   20 |    1 |    18/20 |
```

## 9. Decision Log

이 섹션은 Notion 문서와 PR 설명에 남길 의사결정 기록이다.
구현이 진행되면서 새 판단이 생기면 날짜별로 누적한다.

### 2026-05-27 — Line coverage만으로는 QA 성공 기준이 될 수 없다

- Context: Playwright + Istanbul으로 E2E 실행 code coverage를 수집하는 방식을 검토했다.
- Considered: line coverage를 E2E 자동화 성공 지표로 삼는다.
- Decision: line coverage는 보조 진단 지표로만 둔다.
- Rationale: 한 화면에 진입하기만 해도 layout, table, shared hook, 공통 component가 실행되어 수치가 부풀 수 있다. 실제로 어떤 TC를 검증했는지와 line 실행량은 다르다.
- Metric impact: `line coverage`, `branch coverage`, `function coverage`는 report에 포함하지만 PR blocking 기준으로 단독 사용하지 않는다.

### 2026-05-27 — TC manifest를 상위 source of truth로 둔다

- Context: AI에게 E2E 작성을 위임하려면 “무엇을 검증했는가”를 사람이 만든 답지와 연결해야 한다.
- Considered: Playwright spec 파일 수, test count, line coverage를 기준으로 자동화율을 계산한다.
- Decision: `docs/qa/e2e-tc-manifest.csv`의 TC row를 coverage 분모로 둔다.
- Rationale: TC는 QA가 만든 자연어 시나리오이며, 사람이 검토할 수 있고, 자동화 가능/수동/spec-out 판단을 함께 담을 수 있다.
- Metric impact: `TC automation rate`, `total resolved rate`, `todo`가 최상위 KPI가 된다.

### 2026-05-27 — feasibility와 automation status를 분리한다

- Context: 자동화할 수 없는 TC와 자동화 가능한데 아직 하지 않은 TC를 같은 미완료로 보면 우선순위가 흐려진다.
- Considered: `status` 하나로 `automated`, `manual`, `todo`를 관리한다.
- Decision: `feasibility`와 `automation_status`를 분리한다.
- Rationale: FE E2E 가능 여부는 테스트 작성 상태와 다른 판단이다. POS, 실제 결제, batch, 외부 시스템 의존 TC는 자동화 대상 분모에서 별도 관리해야 한다.
- Metric impact: `automatable`, `manual`, `spec_out`, `blocked`, `needs_review`를 분리해 report한다.

### 2026-05-27 — 첫 PR은 전체 자동화가 아니라 pilot feature 수치화로 제한한다

- Context: E2E coverage instrumentation, TC manifest, report generator, React Doctor integration을 한 번에 도입하면 변경 범위가 넓다.
- Considered: 기존 모든 E2E spec과 모든 QA TC를 한 PR에서 연결한다.
- Decision: 첫 PR은 measurement harness와 pilot feature row 연결까지만 포함한다.
- Rationale: 하네스의 데이터 모델과 report가 맞는지 먼저 확인해야 한다. 전체 spec 전환은 title mapping, fixture migration, evidence policy가 안정화된 뒤 진행한다.
- Metric impact: 첫 PR의 성공 기준은 전체 coverage가 아니라 report 생성 가능성과 pilot feature 수치 출력이다.

### 2026-05-28 — React Doctor는 정적 품질 보조 지표로 통합한다

- Context: AI-generated React code를 검사하는 React Doctor 도입을 검토했다.
- Considered: React Doctor를 ESLint 대체 또는 즉시 blocking gate로 둔다.
- Decision: React Doctor는 ESLint/TypeScript 위에 얹는 advisory static quality layer로 시작한다.
- Rationale: React Doctor는 state/effect, performance, architecture, security, accessibility 문제를 찾는 데 유용하지만, 프로젝트별 false positive와 baseline을 먼저 확인해야 한다.
- Metric impact: `React Doctor score`, `new findings`, `new high findings`, `scanned files`를 QA E2E report에 포함한다.

### 2026-05-28 — 최종 완료 기준은 Notion 문서지만 작성은 구현 후로 미룬다

- Context: repo 문서는 구현 기준과 PR 근거로 필요하지만, 팀 공유와 의사결정 기록은 Notion이 최종 산출물이어야 한다.
- Decision: 구현과 검증이 끝난 뒤 Notion에 “AI QA/E2E Coverage Harness 구축 기록” 문서를 작성하고, 각 레이어별 판단과 지표를 설명하는 것으로 본다.
- Rationale: 이 작업은 단순 코드 변경이 아니라 AI 위임 범위와 agent decision boundary를 정의하는 프로세스 구축이다. 의사결정 맥락이 코드보다 중요하게 재사용된다.
- Completion evidence: Notion 문서 URL, repo standard doc link, pilot report 예시가 있어야 한다.

## 10. 구축 진행 계획

### Phase 0: Raw 기록과 설계 합의

- repo 문서에 raw 의사결정 기록을 먼저 남긴다.
- 참고한 외부 자료, 기존 프로젝트 E2E 전략, React Doctor 도입 의도를 기록한다.
- “무엇을 수치화할 것인가”와 “무엇을 검증할 것인가”를 레이어별로 나눈다.
- 완료 기준: `docs/standards/qa-e2e-coverage-harness.md`에 TC/Run/Code/Static/Evidence 레이어와 의사결정 로그가 설명되어 있다.

### Phase 1: Measurement harness MVP

- TC manifest schema를 추가한다.
- Playwright JSON 결과를 TC manifest와 매핑한다.
- QA E2E report를 생성한다.
- 완료 기준: pilot feature 기준으로 TC automation rate와 E2E pass rate가 나온다.

### Phase 2: E2E code coverage

- `E2E_COVERAGE=1`에서만 Vite instrumentation을 켠다.
- Playwright fixture에서 `window.__coverage__`를 수집한다.
- coverage merge script로 HTML과 summary를 생성한다.
- 완료 기준: `coverage-e2e/coverage-summary.json`이 생성되고 report에 반영된다.

### Phase 3: React Doctor advisory integration

- React Doctor를 opt-in advisory scan으로 실행한다.
- report에 score와 new high findings를 포함한다.
- 기본 실행에서는 `RUN_REACT_DOCTOR=1`이 없으면 skipped report를 남긴다.
- 완료 기준: React Doctor 결과 또는 skipped 사유가 report에 포함되지만 PR을 block하지 않는다.

### Phase 4: Evidence policy

- PASS TC에 screenshot evidence를 연결한다.
- `evidence_required=true`인 TC는 누락 여부를 report한다.
- 완료 기준: pilot feature에서 evidence rate가 계산된다.

### Phase 5: CI warning gate

- PR artifact 또는 comment로 report를 공유한다.
- P0 TC 미매핑, required evidence 누락, new high finding을 warning으로 표시한다.
- 완료 기준: 기존 `yarn test:e2e` 흐름을 깨지 않고 추가 report만 생성한다.

### Phase 6: Notion knowledge capture

- 구현 결과, pilot report, 남은 의사결정 지점을 Notion에 정리한다.
- Notion 위치: `https://www.notion.so/E2E-CI-313b3cce3518805e9e71cba3f5b39c46?source=copy_link`
- 완료 기준: Notion 문서에 각 레이어별 수치화 의도와 검증 기준, agent decision boundary가 기록되어 있다.

## 11. 한 PR 도입 범위

한 PR에 TC 단위 수치화까지 포함할 수 있다.
다만 범위는 전체 TC 자동화가 아니라 측정 하네스와 pilot feature 수치화로 제한한다.

### 포함한다

1. E2E coverage instrumentation
2. coverage 수집 fixture
3. coverage merge script
4. TC manifest schema와 pilot rows
5. Playwright JSON 결과와 TC manifest 매핑
6. QA E2E report 생성 script
7. React Doctor advisory report integration
8. package script
9. `docs/standards/qa-e2e-coverage-harness.md`

### 포함하지 않는다

1. 모든 기존 E2E spec의 coverage fixture 전환
2. 모든 QA TC의 자동화 상태 확정
3. line coverage threshold gate
4. flaky 판정 자동화 고도화
5. writer/critic agent orchestration
6. React Doctor PR gate 승격

### 성공 기준

```txt
yarn test:e2e:coverage
  -> coverage-e2e/coverage-summary.json 생성

yarn qa:e2e:report
  -> .omx/qa/runs/<run-id>/report.md 생성
  -> pilot feature의 TC automation rate, E2E pass rate, evidence rate, line coverage, React Doctor status 출력

기존 yarn test:e2e 실행 방식은 유지
```

## 12. PR Gate 단계

### Phase 1: 측정만 한다

- TC manifest 생성
- Playwright result JSON 수집
- screenshot evidence 수집
- Istanbul coverage report 생성
- React Doctor advisory report 생성
- PR comment 또는 artifact로 report 공유

### Phase 2: 약한 warning gate

- `automatable`인데 `automation_status`가 비어 있으면 warning
- `automated`인데 Playwright result와 매핑되지 않으면 warning
- PASS TC에 required evidence가 없으면 warning
- 변경 파일이 E2E 대상인데 coverage가 0이면 warning
- React Doctor new high finding이 있으면 `advisory-review`

### Phase 3: 강한 TC blocking gate

- P0 automatable TC는 automated 또는 명시적 예외가 있어야 한다.
- P0 automated TC는 PASS해야 한다.
- PASS TC 중 `evidence_required=true`는 evidence가 있어야 한다.
- React Doctor finding은 이 gate에 포함하지 않고 advisory로 분리한다.
- line coverage는 단독 blocking 기준으로 쓰지 않는다.

## 13. Agent Decision Blockers

아래 판단은 에이전트가 제안할 수 있지만 충분한 근거 없이 확정하지 않는다.
근거가 부족하면 작업 완료로 주장하지 않고 `blocked-on-decision` 또는 `advisory-review`로 보고한다.

1. TC가 FE E2E에서 검증 가능한지
2. TC가 최신 기획을 반영하는지
3. 실패 원인이 FE bug, TC mismatch, selector issue, 환경 문제 중 무엇인지
4. `manual`, `spec_out`, `archived` 처리 여부
5. PASS screenshot이 실제 의도한 화면을 검증하는지
6. React Doctor finding을 실제 수정할지, false positive나 deferred item으로 둘지

## 14. 다음 라운드 후보

1. Playwright annotation 기반 `tc_id` 매핑
2. `captureStep(tcId, stepName)` helper 표준화
3. PASS screenshot gallery report
4. changed-file coverage 계산
5. flaky율 계산
6. feature별 worktree/agent 병렬 작성 하네스
7. QA ticket에서 누락 TC를 추출해 manifest에 반영하는 학습 루프
