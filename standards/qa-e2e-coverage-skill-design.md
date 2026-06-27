# QA E2E Coverage Skill Design

> QA E2E coverage harness를 AI 에이전트가 일관되게 실행하고 해석하기 위한 project skill / command 설계

- Status: Draft
- Date: 2026-05-29
- Scope: `.claude/skills/project/qa-e2e-coverage`, `.claude/commands/qa-e2e-coverage`

## 1. 목적

`qa-e2e-coverage` skill은 이미 구축된 QA E2E coverage harness를 에이전트가 안정적으로 사용할 수 있게 만드는 운영 인터페이스다.

현재 harness의 실행 엔진은 package script다.

```txt
yarn test:e2e:coverage
yarn qa:e2e:report
RUN_REACT_DOCTOR=1 yarn test:e2e:coverage
```

하지만 명령어만 있으면 사용자는 매번 아래 판단을 직접 기억해야 한다.

1. 언제 전체 coverage 실행을 해야 하는가
2. 언제 기존 결과만 report로 재생성하면 되는가
3. 언제 React Doctor를 포함해야 하는가
4. report에서 어떤 숫자를 주 지표로 봐야 하는가
5. line coverage를 어느 수준까지 신뢰할 수 있는가
6. `failed`, `unmapped`, `missing evidence`를 어떻게 분류해야 하는가
7. 어떤 항목이 에이전트가 수정 가능한 영역이고, 어떤 항목이 `blocked-on-decision`인지

이 skill의 목적은 위 판단을 에이전트에게 고정하는 것이다.

## 2. 설계 원칙

### 실행 엔진과 운영 인터페이스를 분리한다

package script는 deterministic execution layer로 둔다.
skill은 실행 순서, 해석 규칙, agent gate 판정을 담당한다.

```txt
package scripts = 실행 엔진
project skill = 실행 / 해석 / 결정 경계
command skill = 사용자가 부르기 쉬운 진입점
```

### TC coverage가 주 지표다

skill은 `line coverage`를 성공 기준으로 해석하지 않는다.
주 지표는 `docs/qa/e2e-tc-manifest.csv` 기반의 TC 단위 지표다.

### React Doctor는 advisory layer다

React Doctor는 ESLint/TypeScript 또는 Playwright를 대체하지 않는다.
AI-generated React code의 state/effect, performance, architecture, accessibility, security 신호를 보조로 수집한다.

기본 실행에서 React Doctor가 skipped인 것은 실패가 아니다.
사용자가 React Doctor 포함을 명시하거나 요청 맥락상 React code quality 검토가 필요할 때만 `RUN_REACT_DOCTOR=1`로 실행한다.

### Agent gate를 명확히 분리한다

skill은 실패를 바로 수정하지 않는다.
먼저 아래 중 무엇인지 분류하고, 근거가 부족하거나 작업 범위를 벗어나는 항목은 `blocked-on-decision` 또는 `advisory-review`로 분리한다.

1. FE bug
2. TC mismatch
3. selector / assertion issue
4. env / auth / mock issue
5. spec-out 후보
6. manual 후보
7. React Doctor advisory finding

## 3. 파일 구조

```txt
.claude/
  skills/
    project/
      qa-e2e-coverage/
        SKILL.md
  commands/
    qa-e2e-coverage/
      SKILL.md
docs/
  standards/
    qa-e2e-coverage-harness.md
    qa-e2e-coverage-skill-design.md
```

## 4. Project Skill

### 위치

```txt
.claude/skills/project/qa-e2e-coverage/SKILL.md
```

### Trigger

아래 의도를 가진 요청에서 사용한다.

```txt
QA E2E coverage 봐줘
TC coverage 리포트 확인해줘
E2E coverage harness 실행해줘
QA TC 자동화율 확인해줘
React Doctor도 포함해서 봐줘
failed TC 원인 분류해줘
unmapped TC 확인해줘
missing evidence 확인해줘
```

### Source of truth

skill은 먼저 아래 문서를 읽는다.

```txt
docs/standards/qa-e2e-coverage-harness.md
docs/qa/e2e-tc-manifest.csv
```

필요한 경우 실행 결과를 읽는다.

```txt
.omx/qa/runs/latest/report.md
.omx/qa/runs/latest/tc-results.json
.omx/qa/runs/latest/playwright-results.json
.omx/qa/runs/latest/react-doctor-report.json
.omx/qa/runs/latest/evidence/
coverage-e2e/coverage-summary.json
```

## 5. Modes

### `run`

전체 harness를 실행한다.

사용 상황:

1. 사용자가 “QA E2E coverage 봐줘”라고 요청
2. 최신 결과가 없거나 stale할 수 있음
3. pilot TC / Playwright / coverage / evidence를 다시 생성해야 함

명령:

```bash
yarn test:e2e:coverage
```

출력:

1. TC 지표
2. Runtime 지표
3. Evidence 지표
4. Code coverage 지표
5. React Doctor status
6. Agent gate status

### `report`

기존 실행 산출물로 report만 재생성하고 해석한다.

사용 상황:

1. 사용자가 “기존 결과만 해석해줘”라고 요청
2. E2E 재실행 비용을 피하고 싶음
3. manifest 또는 report script만 수정했음

명령:

```bash
yarn qa:e2e:report
```

### `react-doctor`

React Doctor advisory scan을 포함해 전체 harness를 실행한다.

사용 상황:

1. 사용자가 “React Doctor도 포함”을 명시
2. AI가 React component, hook, state/effect 코드를 수정한 PR
3. E2E는 통과하지만 React 코드 품질 리스크도 같이 보고 싶음

명령:

```bash
RUN_REACT_DOCTOR=1 yarn test:e2e:coverage
```

판정:

1. React Doctor finding은 기본적으로 advisory다.
2. high/critical finding은 `advisory-review`로 분리한다.
3. 작은 안전 수정은 `fix-now`, 범위가 크거나 근거가 부족한 항목은 `blocked-on-decision`으로 분리한다.

### `triage`

최근 report를 읽고 실패/누락/불일치만 분류한다.

사용 상황:

1. `failed TC > 0`
2. `unmapped automated TC > 0`
3. `missing required evidence > 0`
4. React Doctor finding이 존재함

명령:

```bash
yarn qa:e2e:report
```

분류 출력:

| Category               | Meaning                                | Agent action                         | Gate impact                    |
| ---------------------- | -------------------------------------- | ------------------------------------ | ------------------------------ |
| `failed-tc`            | Playwright 실행 실패                   | trace/result 분석                    | `blocked-by-failed-tc`         |
| `unmapped-tc`          | manifest와 Playwright result 연결 실패 | spec path/title 확인                 | `blocked-by-unmapped-tc`       |
| `missing-evidence`     | PASS TC에 required screenshot 없음     | `captureStep` 위치 제안              | `blocked-by-missing-evidence`  |
| `coverage-only-gap`    | TC는 통과했지만 실행 범위가 낮음       | covered files 요약                   | `advisory-only`                |
| `react-doctor-finding` | React Doctor advisory issue            | finding 요약, 수정 가능성 분류      | `advisory-review`              |

## 6. Command Entry

### 위치

```txt
.claude/commands/qa-e2e-coverage/SKILL.md
```

### 책임

command는 긴 정책을 중복하지 않는다.
사용자가 명시적으로 `/qa-e2e-coverage`를 호출할 때 project skill로 라우팅하는 얇은 entrypoint다.

command가 해야 하는 일:

1. `.claude/skills/project/qa-e2e-coverage/SKILL.md`를 사용하라고 지시한다.
2. 사용자의 인자를 mode 후보로 해석한다.
3. 실제 실행/해석 규칙은 project skill에 위임한다.

예:

```txt
/qa-e2e-coverage
/qa-e2e-coverage report
/qa-e2e-coverage react-doctor
/qa-e2e-coverage triage
```

## 7. Output Contract

skill의 최종 응답은 항상 아래 구조를 따른다.

```txt
Mode:
- run | report | react-doctor | triage

Execution:
- command: ...
- result: pass | fail | skipped

Metrics:
- TC total:
- Automatable:
- Automated:
- TC automation rate:
- E2E pass rate:
- Evidence rate:
- Line coverage:
- Branch coverage:
- Function coverage:
- React Doctor status:

Agent gate:
- ...

Artifacts:
- report:
- tc-results:
- coverage:
- evidence:
```

## 8. 해석 규칙

### MUST

1. `line coverage`를 단독 성공 기준으로 말하지 않는다.
2. `TC automation rate`와 `E2E pass rate`를 분리해서 말한다.
3. `unmapped automated TC > 0`이면 성공으로 보고하지 않는다.
4. `missing required evidence > 0`이면 PASS 의미 검토가 불완전하다고 보고한다.
5. React Doctor skipped는 실패가 아니라 advisory 미실행 상태로 설명한다.
6. React Doctor finding은 바로 blocking failure로 단정하지 않고 `advisory-review`로 분리한다.
7. FE bug / TC mismatch / selector issue / env issue 판정 근거가 부족하면 `blocked-on-decision`으로 표시한다.

### SHOULD

1. report 숫자를 그대로 반복하지 말고, 다음 행동이 필요한 항목을 먼저 요약한다.
2. pilot scope인지 전체 scope인지 명시한다.
3. coverage가 높게 나온 경우에도 mount/shared component 영향으로 부풀 수 있음을 언급한다.
4. 새 TC row 추가 또는 spec mapping 변경이 필요한 경우 manifest 수정 후보를 제안한다.

### MUST NOT

1. TC 최신성을 AI가 확정하지 않는다.
2. `manual`, `spec_out`, `archived` 상태를 근거 없이 확정하지 않는다.
3. React Doctor finding을 blocking failure로 단정하지 않는다.
4. `.omx/qa/runs/latest/**`, `.nyc_output/**`, `coverage-e2e/**` 산출물을 커밋 대상으로 제안하지 않는다.

## 9. 사용 예시

### 기본 실행

요청:

```txt
QA E2E coverage 한번 봐줘
```

에이전트:

```txt
yarn test:e2e:coverage 실행
report.md 읽기
TC / Runtime / Evidence / Coverage / React Doctor status 요약
Agent gate status 분리
```

### React Doctor 포함

요청:

```txt
React Doctor도 포함해서 QA E2E coverage 봐줘
```

에이전트:

```txt
RUN_REACT_DOCTOR=1 yarn test:e2e:coverage 실행
React Doctor finding을 advisory로 요약
high/critical finding은 advisory-review로 분리
```

### 실패 triage

요청:

```txt
latest QA E2E report에서 실패 원인 분류해줘
```

에이전트:

```txt
yarn qa:e2e:report 실행
failed / unmapped / missing evidence / React Doctor finding 분리
fix-now / blocked / advisory 항목을 나눔
```

## 10. 완료 기준

1. project skill이 존재한다.
2. command entry가 존재한다.
3. `docs/standards/README.md`에서 이 설계 문서를 찾을 수 있다.
4. skill이 `yarn test:e2e:coverage`, `yarn qa:e2e:report`, `RUN_REACT_DOCTOR=1 yarn test:e2e:coverage`를 올바르게 선택한다.
5. skill이 report를 읽고 TC / Runtime / Evidence / Code coverage / React Doctor를 분리해 요약한다.
6. skill이 수정 가능 항목, blocking 항목, advisory 항목을 `Agent gate`로 분리한다.

## 11. 구현하지 않는 것

이번 설계에서 제외한다.

1. CI PR comment 자동 작성
2. GitHub Actions workflow 추가
3. React Doctor PR gate 승격
4. 모든 E2E spec의 coverage fixture 전환
5. TC manifest 전체 도메인 확장
6. Notion 문서 작성

Notion 문서는 harness와 skill 구현이 끝난 뒤 마지막 knowledge capture 단계에서 작성한다.
