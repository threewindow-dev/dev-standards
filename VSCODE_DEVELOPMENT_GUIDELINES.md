# VS Code 개발 표준

## 목적
- 팀이 동일한 VS Code 환경에서 개발하도록 권장하고, 워크스페이스 구성과 `dev-standards` 노출 방식을 표준화합니다.

## 적용 대상
- 단일 저장소로 개발하는 프로젝트(예: `fastexit-simple`)
- 다중 서비스로 구성된 프로젝트(예: `fastexit`)

## 워크스페이스 유형 요약
- 단일 루트 워크스페이스 (Single-root)
  - 사용 대상: 단일 리포지토리/서비스로 충분한 프로젝트(`{project-name}-simple`).
  - 구성: 리포지토리 루트만 워크스페이스에 추가.
  - `dev-standards` 처리: 리포지토리 루트에 `dev_standards` 서브모듈로 추가(권장). 워크스페이스에서 해당 폴더가 자동으로 노출됨.

- 멀티 루트 워크스페이스 (Multi-root)
  - 사용 대상: 여러 서비스/리포지토리로 구성된 프로젝트(예: `fastexit`).
  - 구성: 각 서비스 리포지토리를 폴더로 추가하고, 상위 워크스페이스 루트도 폴더로 포함하여 `dev-standards`가 보이도록 함.
  - 예: 워크스페이스에 `services/admin-api`, `services/user-web`, 그리고 `workspace-root`(상위 repo)를 추가하고 `dev_standards`를 별도 폴더로 포함시킵니다.
  - 멀티 루트 워크스페이스 규칙: 워크스페이스 폴더 목록에는 항상 먼저 작업공간 루트(`.`)를 추가하고, 그다음 `dev_standards` 폴더를 추가한 뒤에 다른 서비스/서브모듈 폴더를 나열하세요. 이렇게 하면 루트의 설계 문서와 `dev_standards`가 항상 워크스페이스 상단에 노출됩니다.

## `dev-standards` 폴더 전략
- `dev-standards`는 **별도의 최상위 워크스페이스 폴더**로 둡니다.
  - 이유: 중앙화된 규칙·설정·스니펫 접근성 확보, 편리한 업데이트, 워크스페이스 전체에 일관된 노출
  - 구현 방식:
    - 각 개발 리포지토리(서비스)에는 `dev_standards`를 Git 서브모듈로 추가(문서의 기존 규칙과 일치).
    - 멀티 루트 워크스페이스에서는 `dev_standards` 또한 다른 서브모듈처럼 별도 폴더로 추가합니다.
    - 워크스페이스(.code-workspace)에는 각 서비스 폴더와 함께 `.`(워크스페이스 루트)와 `dev_standards` 폴더를 반드시 맨 앞에 명시적으로 포함시켜 UI에 노출되도록 합니다.
  - `dev-standards`는 상위 리포지토리(`{project-name}`)에만 두고 하위 서비스는 상위 리포지토리를 통해 표준을 참조

## 워크스페이스 예시
- 단일 루트 (`fastexit-simple`) — 서브모듈 포함 구조
  - 폴더 구조 예:
    - `fastexit-simple/`
      - `src/`
      - `dev_standards/` (submodule)
      - `.vscode/`

- 멀티 루트 (.code-workspace 예시)
```json
{
  "folders": [
    { "path": "." },
    { "path": "dev_standards" },
    { "path": "services/admin-api" },
    { "path": "services/user-web" },
    { "path": "workspace-root" }
  ],
  "settings": {
    // 워크스페이스 공통 설정
  }
}
```

## 권장 설정 및 확장
- 권장 확장(예): `ESLint`, `Prettier`, `EditorConfig`, `GitLens` 등 팀 표준에 맞춰 목록화
- 워크스페이스 설정 위치
  - 단일 루트: `./.vscode/settings.json`
  - 멀티 루트: `.code-workspace` 파일의 `settings` 또는 각 폴더별 `.vscode/settings.json` 사용

## dev-standards 동기화/업데이트 권장 절차
- 서브모듈로 추가한 경우: 상위/하위 저장소에서 주기적으로 `git submodule update --remote --merge` 실행
- 워크스페이스 레벨에서 `dev_standards`를 항상 보이게 하여 새 규칙/스니펫 반영 여부를 쉽게 확인

## 운영 고려사항
- 접근성: `dev_standards`를 별도 폴더로 두면 검색/참조가 쉬움
- 변경 배포: `dev_standards`를 업데이트하면 사용 저장소(서브모듈)를 업데이트하는 PR 절차를 권장
- 권한/보안: 워크스페이스에 외부 스크립트/설정 포함 시 보안 검토 필수

## 권장 액션 아이템
- 새 프로젝트 시작 시:
  - 단순 프로젝트: `git submodule add <repo-url> dev_standards`
  - 멀티 서비스 프로젝트: 상위 워크스페이스에 `dev_standards` 폴더 추가하고, 각 서비스 저장소에도 서브모듈로 추가(선택)
- 필요 시 `dev_standards`에 공유 VS Code 설정(`settings`, `snippets`, `extensions.json`)을 두어 워크스페이스가 쉽게 일관되도록 구성

---
문서는 팀 상황에 따라 조정 가능합니다. 원하시면 예제 `.code-workspace` 파일을 실제로 생성해 드리겠습니다.
