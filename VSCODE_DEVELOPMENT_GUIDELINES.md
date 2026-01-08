# VS Code 개발 표준

## 목적
- 팀이 동일한 VS Code 환경에서 개발하도록 권장하고, 워크스페이스 구성과 `dev-standards` 노출 방식을 표준화합니다.

## 적용 대상
- 단일 리포지토리 프로젝트 (예: `fastexit-simple`)
- 멀티 리포지토리 프로젝트 (예: `fastexit`)

## 용어 정의

### Git 구조 용어
- **리포지토리(Repository)**: Git으로 버전 관리되는 단일 코드베이스
- **서브모듈(Submodule)**: 다른 리포지토리를 하위 디렉토리로 포함하는 Git 기능
- **단일 리포지토리 프로젝트**: 하나의 리포지토리로 구성된 프로젝트
- **멀티 리포지토리 프로젝트**: 여러 개의 독립된 리포지토리로 구성된 프로젝트 (주로 마이크로서비스 아키텍처)

### VS Code 구조 용어
- **단일 루트 워크스페이스(Single-root Workspace)**: 하나의 폴더를 루트로 하는 VS Code 작업 환경
- **멀티 루트 워크스페이스(Multi-root Workspace)**: 여러 폴더를 동시에 열어 작업하는 VS Code 환경 (`.code-workspace` 파일로 정의)

### 권장 매핑
- **단순 프로젝트**: 단일 리포지토리 = 단일 루트 워크스페이스
- **복잡한 프로젝트**: 멀티 리포지토리 = 멀티 루트 워크스페이스

## 워크스페이스 유형 요약

### 단일 루트 워크스페이스 (Single-root)
- **사용 대상**: 단일 리포지토리 프로젝트 (`{project-name}-simple`)
- **구성**: 리포지토리 루트만 워크스페이스에 추가
- **`dev-standards` 처리**: 리포지토리 루트에 `dev-standards`를 Git 서브모듈로 `.dev-standards` 폴더로 추가(권장). 워크스페이스에서 해당 폴더가 자동으로 노출됨
- **설계 문서**: 리포지토리 루트의 `docs/design/` 폴더에 모든 설계 문서 위치

### 멀티 루트 워크스페이스 (Multi-root)
- **사용 대상**: 멀티 리포지토리 프로젝트 (예: `fastexit`)
- **구성**: 
  - 상위 리포지토리 루트를 첫 번째 폴더로 추가 (공통 설계 문서 포함)
  - `.dev-standards` 폴더를 두 번째 폴더로 추가
  - 각 서비스 리포지토리(Git 서브모듈)를 별도 폴더로 추가
- **멀티 루트 워크스페이스 규칙**: 워크스페이스 폴더 목록에는 항상 먼저 상위 리포지토리 루트(`.`)를 추가하고, 그 다음 `.dev-standards` 폴더를 추가한 뒤 다른 서비스/서브모듈 폴더를 나열하세요. 이렇게 하면 공통 설계 문서와 개발 표준이 항상 워크스페이스 상단에 노출됩니다.
- **설계 문서**: 
  - 공통 설계 문서 (glossary, 유스케이스): 상위 리포지토리 루트의 `docs/design/`
  - 서비스별 API 스펙: 각 서비스 리포지토리의 `docs/design/api-specs/`

## `dev-standards` 폴더 전략
- `dev-standards`는 **별도의 최상위 워크스페이스 폴더**로 둡니다.
  - **이유**: 중앙화된 규칙·설정·스니펫 접근성 확보, 편리한 업데이트, 워크스페이스 전체에 일관된 노출
  - **구현 방식**:
    - 각 리포지토리에는 `dev-standards`를 Git 서브모듈로 추가하고, 서브모듈 폴더명은 `.dev-standards`로 설정합니다
    - 멀티 루트 워크스페이스에서는 `.dev-standards` 폴더를 별도 폴더로 명시적으로 추가합니다
    - 워크스페이스(`.code-workspace`)에는 `.`(상위 리포지토리 루트)와 `.dev-standards` 폴더를 반드시 맨 앞에 포함시켜 UI에 노출되도록 합니다
  - **멀티 리포지토리 프로젝트**: `dev-standards`는 상위 리포지토리(`{project-name}`)에만 두고 하위 서비스 리포지토리는 상위 리포지토리를 통해 표준을 참조

## 프로젝트 유형별 폴더 구조 예시

### 단일 리포지토리 프로젝트 (`fastexit-simple`)
```
fastexit-simple/                 # 리포지토리 루트 = 워크스페이스 루트
├── docs/
│   └── design/                  # 설계 문서 폴더
│       ├── glossary.md          # 도메인 용어 사전
│       ├── use-cases/           # 유스케이스
│       │   ├── business/        # 업무 유스케이스
│       │   └── functional/      # 기능 유스케이스
│       └── api-specs/           # API 스펙
│           └── openapi.yaml
├── src/                         # 소스 코드
├── .dev-standards/              # dev-standards 서브모듈
└── .vscode/
```

### 멀티 리포지토리 프로젝트 (`.code-workspace` 예시)
```
fastexit/                        # 상위 리포지토리 루트
├── docs/
│   └── design/                  # 공통 설계 문서 폴더
│       ├── glossary.md          # 전체 프로젝트 공통 용어
│       └── use-cases/           # 전체 프로젝트 공통 유스케이스
│           ├── business/
│           └── functional/
├── .dev-standards/              # dev-standards 서브모듈
├── services/                    # 서비스 리포지토리들 (Git 서브모듈)
│   ├── admin-api/               # 개별 서비스 리포지토리
│   │   ├── docs/
│   │   │   └── design/
│   │   │       └── api-specs/   # 이 서비스의 API 스펙
│   │   │           └── openapi.yaml
│   │   └── src/
│   ├── user-web/                # 개별 서비스 리포지토리
│   │   ├── docs/
│   │   │   └── design/
│   │   │       └── api-specs/   # 이 서비스의 API 스펙
│   │   │           └── openapi.yaml
│   │   └── src/
│   └── payment-service/
│       ├── docs/
│       │   └── design/
│       │       └── api-specs/
│       │           └── openapi.yaml
│       └── src/
└── .code-workspace              # 멀티 루트 워크스페이스 정의
```

**멀티 루트 워크스페이스 설정 예시 (`.code-workspace`)**:
```json
{
  "folders": [
    { "path": ".", "name": "📁 프로젝트 루트 (공통 문서)" },
    { "path": ".dev-standards", "name": "📋 개발 표준" },
    { "path": "services/admin-api", "name": "🔧 Admin API" },
    { "path": "services/user-web", "name": "🌐 User Web" },
    { "path": "services/payment-service", "name": "💳 Payment Service" }
  ],
  "settings": {
    // 워크스페이스 공통 설정
  }
}
```

## 권장 설정 및 확장
- 권장 확장(예): `ESLint`, `Prettier`, `EditorConfig`, `GitLens` 등 팀 표준에 맞춰 목록화
- Git ignore: 워크스페이스 설정 파일은 개인 설정을 포함하므로 `.gitignore`에 추가해야 합니다. 권장 항목:
  - `.vscode/`
  - `*.code-workspace`
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

### 새 프로젝트 시작 시
- **단일 리포지토리 프로젝트**:
  - `git submodule add <repo-url> .dev-standards` 실행
  - `docs/design/` 폴더 구조 생성 ([DESIGN_DOCUMENT_LOCATION.md](DESIGN_DOCUMENT_LOCATION.md) 참조)
- **멀티 리포지토리 프로젝트**:
  - 상위 리포지토리에 `.dev-standards` 서브모듈 추가
  - 상위 리포지토리에 `docs/design/` 폴더 생성 (공통 문서용)
  - 각 서비스 리포지토리에 `docs/design/api-specs/` 폴더 생성
  - `.code-workspace` 파일 생성 및 폴더 순서 설정
- 필요 시 `.dev-standards`에 공유 VS Code 설정(`settings`, `snippets`, `extensions.json`)을 두어 워크스페이스가 쉽게 일관되도록 구성

## 관련 문서
- [설계 문서 위치 및 폴더 구조 표준](DESIGN_DOCUMENT_LOCATION.md)
- [설계 문서 작성 순서](DESIGN_DOCUMENT_ORDER.md)
- [Git 개발 가이드라인](GIT_DEVELOPMENT_GUIDELINES.md)
