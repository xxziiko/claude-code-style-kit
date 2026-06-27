# 표준 문서

이 디렉터리는 `claude-code-style-kit` 스킬이 직접 참조하는 원본 하네스 표준만 포함한다.

표준 본문은 원본 의도를 보존하기 위해 요약하거나 재작성하지 않는다. 플러그인 범용화는 `skills/`, `agents/`, `commands/`, `README.md`에서 처리한다.

## 포함 표준

- `e2e-network-mocking-standard.md` - Playwright E2E와 MSW browser worker의 network mocking 책임 분리.
- `function-naming.md` - 레이어별 함수 네이밍 규칙.
- `tanstack-query-standard.md` - TanStack Query query/mutation/key/invalidation 적용 기준.
- `test-driven-generation-workflow.md` - 테스트 설계와 테스트 작성 분리, TDD/regression/hardening workflow.
- `typescript-code-style.md` - TypeScript/React import, type import, `any`, re-export 규칙.

## 제외한 원본 표준

아래 문서는 원본 하네스에는 존재하지만 이 플러그인의 핵심 목적에서 제외한다.

- API adoption / REST migration workflow 문서
- REST domain scaffold 설계 문서
- release / QA coverage / CI 운영 하네스 문서
- 프로젝트별 shared UI, table, URL state, Storybook 운영 문서

필요하면 별도 플러그인이나 프로젝트별 overlay로 분리한다.
