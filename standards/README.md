# 표준 문서

이 디렉터리는 Sirloin OMS 프론트엔드 작업에서 에이전트와 구성원이 따라야 하는 현재 구현 표준을 모은다.

## 확정 표준

- `api-domain-ui-standard.md` - API, domain, UI 레이어 경계.
- `component-composition-standard.md` - 서브컴포넌트 분리/조합 규칙과 조건부 렌더링 안티패턴.
- `component-storybook-standard.md` - 컴포넌트와 Storybook 사용 표준.
- `domain-feature-architecture-standard.md` - 목표 domain/feature 아키텍처와 변경 파일 감사 규칙.
- `function-naming.md` - 레이어별 함수 네이밍 규칙.
- `project-harness-standard.md` - 프로젝트 공유 하네스 경계와 검증 표준.
- `shared-ui-conventions.md` - shared UI, hook, util, constant 배치와 public API 표준.
- `table-scroll-policy.md` - 테이블 가로스크롤 UX와 예외 테이블 정책.
- `tanstack-query-standard.md` - React Query 사용 규칙.
- `typescript-code-style.md` - TypeScript 코드 스타일 규칙 (React import, any 금지, re-export 금지).

## 현재 워크플로우 계약

아래 문서는 헤더에 draft 상태가 남아 있을 수 있지만, 더 최신 표준으로 대체되기 전까지 이 프로젝트의 현재 운영 계약으로 취급한다.

- `automation-boundary.md` - 스크립트가 자동 생성해도 되는 범위와 사람/에이전트 판단이 필요한 범위.
- `e2e-ci-strategy.md` - Playwright E2E 구조와 CI 선택 실행 전략.
- `e2e-network-mocking-standard.md` - Playwright E2E 네트워크 mocking과 dialog assertion 표준.
- `qa-e2e-coverage-harness.md` - QA TC 단위 E2E coverage 측정과 리포트 하네스.
- `qa-e2e-coverage-skill-design.md` - QA E2E coverage 하네스를 운용하기 위한 에이전트 스킬/커맨드 설계.
- `release-strategy.md` - semantic-release 기반 브랜치별 릴리스 전략.
- `rest-api-migration-standard.md` - GraphQL에서 REST로 전환할 때의 마이그레이션 규칙.
- `create-api-workflow-standard.md` - create-api workflow의 현재 source of truth.
- `scaffold-rest-domain-design.md` - REST domain scaffold 설계와 제약.
- `test-driven-generation-workflow.md` - generated/API 기반 작업의 test-first workflow.

## 관련 Source Of Truth

- `docs/standards/function-naming.md` - 레이어별 함수 네이밍 규칙.
- `AGENTS.md` - 프로젝트 공유 컨벤션과 표준 계약.
- `CLAUDE.md` - Claude Code용 프로젝트 컨텍스트.
- `README.md` - 프로젝트 설정, 아키텍처 요약, 공통 명령어.
- `.claude/commands/*/SKILL.md` - 프로젝트 공유 Claude slash-command 진입점.
- `.claude/skills/project/*/SKILL.md` - 프로젝트 표준과 scaffold workflow를 실행 가능한 형태로 보강하는 skill overlay.

## 유지보수 규칙

- 현재 구현에 영향을 주는 규칙만 active standards에 둔다.
- 완료된 계획, 대체된 설계 초안, 과거 reference audit, 로컬 하네스 전략 메모는 이 디렉터리 밖에 둔다.
- 과거 reference audit은 `docs/history/reference-audits/` 또는 로컬 `_harness/history/` 트리에 둔다.
- 같은 규칙을 여러 파일에 중복 작성하기보다 하나의 active source of truth로 연결한다.
- 개인/로컬 하네스 동작은 이 프로젝트 저장소 밖에 둔다.
- 프로젝트 공유 command와 skill overlay는 `.claude/`에 둔다.
- 개인 에이전트 persona와 orchestration preference는 이 프로젝트 저장소 밖에 둔다.
