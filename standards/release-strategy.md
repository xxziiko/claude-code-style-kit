# Release Strategy

## 개요

이 프로젝트는 [semantic-release](https://semantic-release.gitbook.io/)를 사용해 **Conventional Commits** 기반으로 버전을 자동 관리합니다.

배포는 `develop` / `main` 브랜치에 대한 **push 트리거**로 자동 실행됩니다.

---

## 브랜치 전략

| 브랜치 | 유형 | 태그 예시 | CHANGELOG | package.json | Release Commit |
|--------|------|---------|-----------|-------------|----------------|
| `develop` | prerelease | `v1.2.3-dev.1` | ✅ | ✅ | `chore(release): 1.2.3-dev.1 [skip ci]` |
| `main` | stable | `v1.2.3` | ✅ | ✅ | `chore(release): 1.2.3 [skip ci]` |

- staging 브랜치 전략은 사용하지 않습니다. `.env.staging`은 환경 파일로만 존재합니다.

---

## 워크플로 구조

```
.github/workflows/
├── release-develop.yml        # develop push → prerelease
├── release-main.yml           # main push → stable release + Notion 로그 + back-merge
├── ci.yml                     # PR 검증 (standards check + typecheck + build)
├── e2e-nightly.yml            # 야간 전체 E2E (평일 KST 00:00)
└── pr-slack-notification.yml  # PR Slack 알림
```

설정 파일:
- `release.config.develop.cjs` — develop prerelease 설정
- `release.config.main.cjs` — main stable 설정

Notion 로깅 스크립트: `scripts/notion-deploy-log.sh` (release-main.yml에서 호출)

---

## 운영 플로우

### dev 배포

1. `develop` 브랜치에 PR 머지 (push)
2. `release-develop.yml` 자동 실행: semantic-release (prerelease)
3. 릴리즈 트리거 커밋이 있으면 `v1.2.3-dev.N` 태그 생성 + GitHub Prerelease 발행

### production 배포

1. `main` 브랜치에 PR 머지 (push)
2. `release-main.yml` 자동 실행:
   - `release` job: semantic-release (stable)
     - `v1.2.3` 태그 생성
     - `CHANGELOG.md` + `package.json` 갱신
     - `chore(release): 1.2.3 [skip ci]` 커밋 push
     - GitHub Release 발행
     - Notion 배포 로그 기록 (`scripts/notion-deploy-log.sh`)
   - `back-merge` job: release가 실제로 발행된 경우에만 실행
     - `main → develop` 자동 백머지 (`[skip ci]`)

### 동작 흐름 예시

```
# dev 배포
push → develop
→ release-develop.yml
→ tag: v3.18.0-dev.1
→ GitHub Prerelease 발행

# production 배포
push → main
→ release-main.yml (release job)
  → tag: v3.18.0
  → CHANGELOG.md / package.json 갱신
  → chore(release): 3.18.0 [skip ci]
  → GitHub Release 발행
  → Notion 배포 로그 기록
→ release-main.yml (back-merge job)
  → main → develop 백머지 [skip ci]
```

---

## 커밋 타입 → 버전 bump 규칙

[Conventional Commits](https://www.conventionalcommits.org/) 규칙을 따릅니다.

| 커밋 타입 | 버전 bump | 예시 커밋 |
|----------|----------|---------|
| `feat:` | **minor** (1.2.x → 1.3.0) | `feat: 재고실사 Drawer 기능 추가` |
| `fix:` | **patch** (1.2.3 → 1.2.4) | `fix: 바코드 검색 오류 수정` |
| `perf:` | **patch** | `perf: 재고 목록 쿼리 최적화` |
| `BREAKING CHANGE:` (footer) | **major** (1.2.3 → 2.0.0) | `feat!: API 응답 구조 변경` |
| `chore:`, `docs:`, `style:`, `refactor:`, `test:`, `ci:`, `build:` | **릴리즈 없음** | — |

> **주의**: 프로젝트 custom type(`Add`, `Update`, `Hotfix`, `Remove`)은 semantic-release에서 인식되지 않습니다.
> 릴리즈를 트리거하려면 `feat:` 또는 `fix:`를 사용하세요.

---

## Release Loop 방지

- semantic-release가 push하는 release commit 메시지에 `[skip ci]`가 포함됩니다.
- back-merge 커밋 메시지에도 `[skip ci]`가 포함됩니다.
- `[skip ci]`가 포함된 커밋은 GitHub Actions가 워크플로우를 트리거하지 않으므로 무한 루프가 발생하지 않습니다.

---

## Git Identity

- **release 커밋/태그**: semantic-release 공식 기본값 (`semantic-release-bot <semantic-release-bot@martynus.net>`)
  - workflow에서 `GIT_AUTHOR_*` / `GIT_COMMITTER_*` env를 설정하지 않음
- **back-merge 커밋**: `github-actions[bot]` (수동 merge라 맥락 분리)
- 실제 push 인증은 `G_TOKEN`(PAT) 기준

---

## 필요한 GitHub Secrets

| Secret | 용도 | 필요 워크플로 |
|--------|------|------------|
| `G_TOKEN` | protected branch push (release commit, back-merge) | `release-develop.yml`, `release-main.yml` |
| `NOTION_TOKEN` | Notion API 인증 | `release-main.yml` |
| `NOTION_DATABASE_ID` | Notion 배포 로그 DB | `release-main.yml` |

---

## 초기 설정 (최초 1회)

semantic-release는 `v` 접두사 태그를 기준으로 버전을 계산합니다.

```bash
# 현재 최신 버전 확인
git tag --sort=-v:refname | head -5

# main HEAD에 v 접두사 태그 생성 (예: 현재 최신이 3.17.16인 경우)
git tag v3.17.16 main
git push origin v3.17.16
```

---

## dry-run 검증

```bash
# develop prerelease dry-run
GITHUB_TOKEN=<token> npx semantic-release --dry-run --extends ./release.config.develop.cjs

# main stable dry-run
GITHUB_TOKEN=<token> npx semantic-release --dry-run --extends ./release.config.main.cjs
```
