# 설계 문서 위치 및 폴더 구조 표준

## 목적
프로젝트의 설계 문서가 저장되는 위치와 폴더 구조를 명확히 정의하여, 팀원 누구나 일관된 방식으로 문서를 찾고 관리할 수 있도록 합니다.

## 용어 정의

### Git 구조 용어
- **리포지토리(Repository)**: Git으로 버전 관리되는 단일 코드베이스
- **서브모듈(Submodule)**: 다른 리포지토리를 하위 디렉토리로 포함하는 Git 기능
- **멀티 리포지토리 프로젝트**: 여러 개의 독립된 리포지토리로 구성된 프로젝트

### VS Code 구조 용어
- **단일 루트 워크스페이스(Single-root Workspace)**: 하나의 폴더를 루트로 하는 VS Code 작업 환경
- **멀티 루트 워크스페이스(Multi-root Workspace)**: 여러 폴더를 동시에 열어 작업하는 VS Code 환경 (`.code-workspace` 파일로 정의)

### 권장 매핑
- **단순 프로젝트**: 단일 리포지토리 = 단일 루트 워크스페이스
- **복잡한 프로젝트**: 멀티 리포지토리 = 멀티 루트 워크스페이스

## 설계 문서 폴더 구조

### 기본 원칙
1. 모든 설계 문서는 `docs/design/` 폴더 아래에 위치합니다
2. 폴더명은 소문자와 하이픈을 사용합니다 (예: `use-cases`, `api-specs`)
3. 파일명은 명확하고 설명적이어야 하며, 하이픈으로 단어를 구분합니다

### 표준 폴더 구조

```
{project-root}/
├── docs/
│   └── design/
│       ├── glossary.md           # 도메인 용어 사전
│       ├── use-cases/            # 유스케이스 문서
│       │   ├── business/         # 업무 유스케이스
│       │   │   ├── UC-B001-user-registration.md
│       │   │   └── UC-B002-payment-process.md
│       │   └── functional/       # 기능 유스케이스
│       │       ├── UC-F001-api-authentication.md
│       │       └── UC-F002-data-validation.md
│       └── api-specs/            # API 명세서
│           ├── openapi.yaml      # 통합 OpenAPI 스펙
│           └── endpoints/        # 엔드포인트별 세부 문서 (선택)
│               ├── users.yaml
│               └── payments.yaml
```

## 프로젝트 유형별 설계 문서 위치 규칙

### 1. 단일 리포지토리 프로젝트

**적용 대상**: `{project-name}-simple` 형태의 단순 프로젝트

**설계 문서 위치**:
- **Glossary**: 리포지토리 루트의 `docs/design/glossary.md`
- **유스케이스**: 리포지토리 루트의 `docs/design/use-cases/`
- **API 스펙**: 리포지토리 루트의 `docs/design/api-specs/`

**폴더 구조 예시**:
```
fastexit-simple/                 # 리포지토리 루트 = 워크스페이스 루트
├── docs/
│   └── design/
│       ├── glossary.md
│       ├── use-cases/
│       │   ├── business/
│       │   └── functional/
│       └── api-specs/
│           └── openapi.yaml
├── src/                         # 소스 코드
├── .dev-standards/              # dev-standards 서브모듈
└── .vscode/
```

### 2. 멀티 리포지토리 프로젝트

**적용 대상**: 여러 서비스/마이크로서비스로 구성된 복잡한 프로젝트

**설계 문서 위치**:
- **Glossary**: 상위 리포지토리 루트의 `docs/design/glossary.md` (전체 프로젝트 공통)
- **유스케이스**: 상위 리포지토리 루트의 `docs/design/use-cases/` (전체 프로젝트 공통)
- **API 스펙**: 각 서비스 리포지토리의 `docs/design/api-specs/` (서비스별로 분리)

**폴더 구조 예시**:
```
fastexit/                        # 상위 리포지토리 (워크스페이스 루트)
├── docs/
│   └── design/
│       ├── glossary.md          # 전체 프로젝트 공통 용어
│       └── use-cases/           # 전체 프로젝트 공통 유스케이스
│           ├── business/
│           └── functional/
├── .dev-standards/              # dev-standards 서브모듈
├── admin-api/               # 개별 서비스 리포지토리 (Git 서브모듈)
│   ├── docs/
│   │   └── design/
│   │       └── api-specs/   # 이 서비스의 API 스펙
│   │           └── openapi.yaml
│   └── src/
├── user-web/                # 개별 서비스 리포지토리  (Git 서브모듈)
│   ├── docs/
│   │   └── design/
│   │       └── api-specs/   # 이 서비스의 API 스펙
│   │           └── openapi.yaml
│   └── src/
└── payment-service/         # 개별 서비스 리포지토리  (Git 서브모듈)
│   ├── docs/
│   │   └── design/
│   │       └── api-specs/   # 이 서비스의 API 스펙
│   │           └── openapi.yaml
│   └── src/
└── .code-workspace              # 멀티 루트 워크스페이스 정의
```

### 3. 공통 vs 서비스별 문서 배치 원칙

| 문서 유형 | 단일 리포지토리 | 멀티 리포지토리 | 이유 |
|---------|--------------|--------------|------|
| **Glossary** | 루트 `docs/design/` | 상위 리포지토리 루트 `docs/design/` | 도메인 용어는 프로젝트 전체에서 공유 |
| **유스케이스** | 루트 `docs/design/use-cases/` | 상위 리포지토리 루트 `docs/design/use-cases/` | 비즈니스/기능 요구사항은 프로젝트 전체 관점에서 관리 |
| **API 스펙** | 루트 `docs/design/api-specs/` | 각 서비스 리포지토리의 `docs/design/api-specs/` | API는 서비스별로 독립적으로 버전 관리 |

## 파일 명명 규칙

### Glossary
- 파일명: `glossary.md` (고정)
- 위치: `docs/design/glossary.md`

### 유스케이스
- 업무 유스케이스: `UC-B{번호}-{제목}.md`
  - 예: `UC-B001-user-registration.md`
- 기능 유스케이스: `UC-F{번호}-{제목}.md`
  - 예: `UC-F001-api-authentication.md`
- 위치: `docs/design/use-cases/business/` 또는 `docs/design/use-cases/functional/`

### API 스펙
- OpenAPI 파일: `openapi.yaml` 또는 `openapi-{버전}.yaml`
- 추가 스펙: `{리소스명}-api.yaml` 또는 `{기능명}-spec.yaml`
- 위치: `docs/design/api-specs/`

## VS Code 워크스페이스 설정

### 단일 루트 워크스페이스
- 리포지토리 루트를 VS Code에서 직접 열기
- `docs/design/` 폴더가 자동으로 보임

### 멀티 루트 워크스페이스
- `.code-workspace` 파일에서 폴더 순서 권장:
  1. `.` (상위 리포지토리 루트 - 공통 설계 문서 포함)
  2. `.dev-standards` (개발 표준)
  3. 각 서비스 폴더들

**예시 `.code-workspace`**:
```json
{
  "folders": [
    { "path": ".", "name": "📁 프로젝트 루트" },
    { "path": ".dev-standards", "name": "📋 개발 표준" },
    { "path": "services/admin-api", "name": "🔧 Admin API" },
    { "path": "services/user-web", "name": "🌐 User Web" }
  ]
}
```

## 체크리스트

### 프로젝트 시작 시
- [ ] `docs/design/` 폴더 생성
- [ ] `glossary.md` 생성 (템플릿: `.dev-standards/templates/glossary_template.md`)
- [ ] `use-cases/business/` 폴더 생성
- [ ] `use-cases/functional/` 폴더 생성
- [ ] `api-specs/` 폴더 생성
- [ ] (멀티 리포지토리) 각 서비스에 `docs/design/api-specs/` 폴더 생성

### 문서 작성 시
- [ ] [DESIGN_DOCUMENT_ORDER.md](DESIGN_DOCUMENT_ORDER.md)의 작성 순서 준수
- [ ] 템플릿 사용 (business_use_case_template.md, functional_use_case_template.md 등)
- [ ] 문서 간 링크 시 상대 경로 사용
- [ ] API 스펙 작성 후 Spectral 린트 실행

## 관련 문서
- [설계 문서 작성 순서](DESIGN_DOCUMENT_ORDER.md)
- [API 스펙 표준](API_SPEC_STANDARDS.md)
- [유스케이스 표준](USE_CASE_STANDARDS.md)
- [VS Code 개발 표준](VSCODE_DEVELOPMENT_GUIDELINES.md)
- [Git 개발 가이드라인](GIT_DEVELOPMENT_GUIDELINES.md)
