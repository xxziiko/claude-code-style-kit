---
name: commit
category: project
description: 커밋 컨벤션에 맞는 커밋 생성. "커밋해줘", "커밋 만들어줘", "변경사항 커밋" 요청 시 사용.
command: /commit
---

# Commit Skill

프로젝트의 Conventional Commits 규칙에 맞춰 변경사항을 분석하고, 적절한 커밋 메시지를 작성하여 커밋합니다.

## 커밋 컨벤션

```text
<type>(<scope>): <subject>

[body]

[footer]
```

### type (필수)

| type | 설명 | 예시 |
|---|---|---|
| `feat` | 새로운 기능 추가 | `feat: 목록 필터 개선` |
| `fix` | 버그 수정 | `fix(ui): 수량 검증 오류 수정` |
| `refactor` | 기능 변화 없는 구조 개선 | `refactor: 필터 훅 분리` |
| `docs` | 문서/주석 수정 | `docs: README 커밋 규칙 추가` |
| `style` | 포맷/스타일 정리 (동작 변경 없음) | `style: import 정렬 정리` |
| `test` | 테스트 추가/수정 | `test: 필터 훅 테스트 보강` |
| `chore` | 설정/도구/의존성 등 기타 작업 | `chore: husky 커밋 훅 정비` |
| `ci` | CI/CD 설정 변경 | `ci: Playwright 워크플로우 조건 수정` |
| `build` | 빌드 시스템/외부 의존성 변경 | `build: vite 설정 업데이트` |
| `perf` | 성능 개선 | `perf: 목록 가상 스크롤 렌더링 최적화` |
| `revert` | 이전 커밋 되돌리기 | `revert: 필터 초기화 동작 복원` |

### scope (선택)

- 영향 범위를 좁혀 설명하고 싶을 때만 사용
- 예: `auth`, `ui`, `test`, `config`
- 형식: `type(scope): subject`

### Subject (필수)

- 한글 작성 권장
- 마침표 없이 작성
- 명령형으로 작성 (예: "추가", "수정", "개선", "제거")
- 짧고 명확하게 핵심 변경 의도를 표현

### Body / Footer (선택)

- 변경 이유와 맥락 설명이 필요한 경우 body를 작성
- body는 subject 다음 빈 줄로 분리
- 호환성 깨짐이 있으면 `type(scope)!: subject` 형식 또는 footer의 `BREAKING CHANGE:`를 사용
- 관련 이슈/참조가 있으면 `Refs: #123` 같은 trailer를 추가 가능

## 워크플로우

### 1. 변경사항 분석 (병렬 실행)

```bash
git status
git diff --staged
git diff
git log --oneline -5
```

### 2. 스테이징 확인

**이미 staged된 파일이 있는 경우:**
- staged된 파일만 커밋 대상으로 사용
- "현재 staged된 파일로 커밋할까요?" 확인

**staged된 파일이 없는 경우:**
- unstaged 변경 파일 목록 제시
- 사용자에게 질문: "어떤 파일을 커밋할까요?"
- 선택지: `전체`, `변경된 파일 목록 제시`
  → 답변에 따라 `git add` 실행

**주의:**
- `.env`, `credentials`, 시크릿 파일은 커밋 대상에서 제외
- untracked 파일은 별도로 안내

### 3. 커밋 메시지 작성

변경사항을 분석하여 커밋 메시지 초안을 작성합니다.

**type 판단 기준:**

| 변경 패턴 | type |
|---|---|
| 새 기능 추가 (컴포넌트, 훅, 화면, 플로우 등) | `feat` |
| 버그 수정 | `fix` |
| 코드 구조 개선 (기능 변경 없음) | `refactor` |
| 문서/주석 변경 | `docs` |
| 포맷/스타일만 수정 (동작 변경 없음) | `style` |
| 테스트 추가/수정 | `test` |
| 설정, 훅, 스크립트, 의존성 변경 | `chore`, `ci`, `build` |
| 성능 개선 | `perf` |
| 기존 커밋 되돌리기 | `revert` |

**복합 변경 시:**
- 새 파일 추가 + 기존 파일 수정 → 주요 변경 의도에 따라 결정
- 사용자 가치가 늘어나는 변경이 주이면 `feat`
- 버그 해결이 주이면 `fix`
- 구조 개선이 주이면 `refactor`

### 4. 사용자 확인

```text
## 커밋 메시지 초안

{type}({scope}): {subject}

{body / footer (있을 경우)}

---
스테이징 파일:
- {파일 1}
- {파일 2}

이 메시지로 커밋할까요?
```

→ **사용자 승인 필수 — 승인 없이 커밋 금지**

### 5. 커밋 실행

```bash
git commit -m "$(cat <<'EOF'
{커밋 메시지}
EOF
)"
```

### 6. 결과

```text
✅ 커밋 완료

커밋: {hash} {message}
변경 파일: {N}개

push가 필요하면 알려주세요.
```

## 주의

- `--no-verify` 사용 금지
- `--amend` 사용 시 사용자에게 명시적 확인
- force push 절대 금지
- 시크릿 파일 커밋 방지
- 커밋 메시지는 반드시 HEREDOC으로 전달 (특수문자 안전)
- 새 커밋은 Conventional Commits 1.0.0 규칙(`type(scope): subject`)을 따른다
