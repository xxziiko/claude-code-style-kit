# Xxziiko

실무 작업을 `scan -> do -> fix -> check` 흐름으로 진행하기 위한 Claude Code 플러그인입니다.

코드 스타일, 테스트 설계/작성, 리뷰, 검증, TanStack Query 규칙, 커밋 워크플로우를 함께 제공합니다.

## 바로 설치

Claude Code에서 아래 명령을 실행합니다.

```text
/plugin marketplace add xxziiko/claude-code-style-kit
/plugin install xxziiko@xxziiko-style-kit
```

공개 repository라 GitHub read 인증 없이 설치할 수 있습니다.

Repository: https://github.com/xxziiko/claude-code-style-kit

## 기본 사용 흐름

```text
/xxziiko:scan
/xxziiko:do
/xxziiko:fix
/xxziiko:check
```

- `/xxziiko:scan`: 수정 없이 프로젝트 구조, 요구사항, 성공 기준, 기존 패턴, 구현 계획을 파악합니다.
- `/xxziiko:do`: 기존 패턴을 우선하고 새 의존성 없이 최소 변경으로 구현합니다.
- `/xxziiko:fix`: 에러 로그와 재현 증거를 기반으로 원인을 분석하고 최소 수정합니다.
- `/xxziiko:check`: 변경 요약, 검증 결과, 남은 리스크를 정리합니다.

## 포함된 명령

```text
/xxziiko:scan
/xxziiko:do
/xxziiko:fix
/xxziiko:check
/xxziiko:test-plan
/xxziiko:write-tests
/xxziiko:review
/xxziiko:verify
/xxziiko:commit
```

## 포함된 내용

- TypeScript/React 코드 스타일 기준
- 함수 네이밍 규칙
- DTO/domain boundary 규칙
- TanStack Query query/mutation 패턴
- 테스트 설계와 테스트 작성 워크플로우
- 코드 리뷰, 검증, 빌드 수정, 아키텍처 리뷰 에이전트
- `scan/do/fix/check` 실무 실행 흐름

## 함께 쓰기 좋은 플러그인

이 플러그인은 중심 흐름을 담당합니다. 외부 플러그인은 특정 관점의 보조 렌즈로 사용하는 것을 권장합니다.

- `ponytail`: 과설계 방지, 삭제 우선, 최소 diff 압박
- `frontend-fundamentals`: 프론트엔드 품질, 접근성, 컴포넌트 책임 점검
- `oh-my-claudecode`: 큰 작업에서 병렬 에이전트 운용

조합 예시는 `COMPANIONS.md`를 참고하세요.

## 로컬 테스트

Claude CLI가 설치되어 있다면 repository root에서 실행합니다.

```bash
claude --plugin-dir .
```

그 다음 아래 명령을 확인합니다.

```text
/xxziiko:scan
/xxziiko:check
```

## 포함하지 않는 것

- 특정 회사/프로젝트 전용 API adoption workflow
- 외부 플러그인 자동 설치
- 회사별 브랜치, PR, 릴리즈 규칙

