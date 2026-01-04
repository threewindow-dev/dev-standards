# Git 개발 표준

## 1. 목적
- 팀 내 일관된 Git 사용 방식 제공
- 코드 변경 이력의 가독성, 추적성, 협업 효율 향상

## 2. 적용 범위
- 모든 애플리케이션, 라이브러리, 인프라 관련 저장소

## 3. 브랜칭 모델
- 기본 브랜치
  - `main`: 항상 배포 가능한 상태 유지
  - `dev`: 1인 개발시, 개발 브랜치 
- 기능 브랜치
  - 패턴: `feature/<짧은-설명>`
  - 예: `feature/login-rate-limit`
- 릴리즈/핫픽스
  - `release/<version>`: 배포 준비
  - `hotfix/<issue>`: 긴급 수정

## 4. 브랜치 네이밍 규칙
- 모두 소문자, 단어는 `-`로 구분
- 이슈 번호가 있을 경우 접미사로 포함 가능: `feature/123-add-login`

## 5. 작업 흐름
- 기본: 브랜치 생성 → 작업 → 커밋 → PR 생성 → 코드리뷰 → 병합
- 병합 전략
  - `main`으로는 보호 규칙 적용(필수 PR, CI 통과)
  - 병합 방식: 팀 합의에 따라 `squash` 또는 `merge commit` 사용
- Rebase vs Merge
  - 개인 로컬 정리는 `rebase` 권장
  - 공개된 브랜치에 대해서는 `merge`를 선호

## 6. 커밋 메시지 규약 (Conventional Commits 기반)
- 형식: `type(scope): subject`
  - `type`: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore` 등
  - `scope`: 선택적(모듈, 컴포넌트)
  - `subject`: 명령형 현재 시제, 50자 이내 권장
  - 본문은 필요시 빈 줄 후 상세 설명(72자 줄바꿈 권장)
  - 예: `feat(auth): add refresh token support`

## 7. Pull Request (PR) 규칙
- PR 템플릿을 사용하여 목적, 변경사항, 테스트 방법 명시
- 최소 리뷰어 1명 이상 지정
- PR 크기 권장: 작은 단위(가능하면 200라인 미만)
- 머지 전 CI(테스트/린트/빌드) 통과 필수
- 관련 이슈 번호 연결 및 필요시 스크린샷/로그 첨부

## 8. 민감 정보 및 보안
- 비밀번호, API 키, 인증 정보는 절대 커밋하지 않음
- 환경 변수, 시크릿은 시크릿 관리 도구(예: Vault, GitHub Secrets) 사용
- 실수로 민감 정보 커밋 시 즉시 리포지토리 관리자에게 보고하고 `git filter-repo` 등으로 제거

## 9. .gitignore 및 파일 관리
- 각 프로젝트에 적절한 `.gitignore` 유지
- 빌드 아티팩트, 로그, 개인 설정 파일은 추적하지 않음

## 10. Git Hooks 및 자동화
- `pre-commit`으로 포맷/린트 자동 실행 권장
- CI에서 강제 검사(포맷, 린터, 테스트)를 설정

## 11. 태그 및 릴리즈
- 배포 시점에 의미 있는 태그 사용: `v1.2.3`
- 태그는 반드시 `main`의 배포 가능한 커밋에 붙임
- 릴리즈 노트에 변경 요약 작성

## 12. 커밋 서명
- 보안 정책에 따라 커밋 서명(`git commit -S`)을 요구할 수 있음

## 13. 충돌 해결 가이드
- 최신 `main`을 로컬에 병합/리베이스 후 충돌 해결
- 충돌 해결 시 테스트 실행으로 동작 확인

## 14. 유용한 예시 명령
- 브랜치 생성: `git checkout -b feature/xxx`
- 변경 스테이지: `git add .` 또는 `git add <file>`
- 커밋: `git commit -m "type(scope): subject"`
- 최신 반영(rebase): `git fetch origin && git rebase origin/main`
- PR 생성 후 머지(예시): `git checkout main && git pull && git merge --no-ff feature/xxx && git push`
 - 서브모듈로 `dev_standards` 추가: `git submodule add <repo-url> dev_standards`
 - 서브모듈 초기화/업데이트: `git submodule update --init --recursive`
 - 원격 서브모듈을 최신으로 당기기: `git submodule update --remote --merge`

## 15. 시행 및 준수
- 이 표준은 팀 합의 하에 정기적으로 업데이트
- CI/저장소 보호 규칙으로 주요 규칙을 강제 적용
 - 각 프로젝트 루트 리포지토리에는 `dev_standards` 저장소를 서브모듈로 `dev_standards` 폴더에 추가해야 합니다. 예:
   - `git submodule add https://github.com/threewindow-dev/dev-standards.git dev_standards`
   - 서브모듈 변경 사항은 별도의 커밋/PR로 관리하고, 필요시 주기적으로 업데이트하세요.

## 16. 레포지토리 네이밍 규칙
- 목적: 프로젝트 구조가 명확히 드러나도록 이름 규칙을 통일합니다.
- 기본 원칙
  - 모두 소문자, 단어는 `-`로 구분
  - 공통 접두사로 `project-name`을 사용하여 관련 리포지토리 그룹화

- 단일 저장소로 개발하는 경우 (간단한 프로젝트)
  - 패턴: `{project-name}-simple`
  - 예: `fastexit-simple`
  - 설명: 서비스가 하나의 리포지토리로 충분할 때 사용

- 다중 컴포넌트(마이크로서비스/멀티 리포지토리) 방식
  - 패턴: `{project-name}-{service-name}-{type}`
  - 예: `fastexit-admin-api`, `fastexit-user-web`, `fastexit-payments-worker`
  - `type` 예시: `api`, `web`, `worker`, `db`, `infra` 등

- 상위(모노) 레포지토리와 서브모듈 연결
  - 상위 리포지토리 패턴: `{project-name}` (예: `fastexit`)
  - 구성: 상위 리포지토리는 하위 서비스 리포지토리들을 서브모듈로 포함하여 전체 프로젝트 관리를 쉽게 함
  - 예시 폴더 구조 (상위 리포지토리 `fastexit`):
    - `services/admin-api` -> 서브모듈 pointing to `fastexit-admin-api`
    - `services/user-web` -> 서브모듈 pointing to `fastexit-user-web`
    - `libs/shared-auth` -> 서브모듈 pointing to `fastexit-shared-auth` (선택)

- 서브모듈 연결 권장 절차
  - 각 하위 리포지토리에서 이름 규칙 준수 후 퍼블리시
  - 상위 리포지토리에서 아래와 같이 추가:
    - `git submodule add https://github.com/<org>/fastexit-admin-api.git services/admin-api`
    - `git submodule add https://github.com/<org>/fastexit-user-web.git services/user-web`
  - 상위 리포지토리는 각 서브모듈 버전(커밋)을 고정하여 안정적 빌드를 보장

- 권장사항
  - 리포지토리 이름에 공백, 특수문자 사용 금지
  - 서비스 영역이 명확하도록 `service-name`은 짧고 설명적으로 작성
  - 공통 라이브러리/헬퍼는 `libs/` 또는 `shared-` 접두사로 구분


## 16. 참고 자료
- Conventional Commits: https://www.conventionalcommits.org
- Git 브랜칭 모델 참고: Git Flow, GitHub Flow 등

---
본 문서는 팀의 필요에 따라 수정 가능합니다. 변경 제안은 PR로 제출하세요.
