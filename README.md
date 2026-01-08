threewindow-dev의 github 저장소에서 관리되는 개발프로젝트에 적용되는 개발 표준입니다.

## 표준 문서 목록

### 설계 문서 관련
- [설계 문서 위치 및 폴더 구조 표준](DESIGN_DOCUMENT_LOCATION.md) - 설계 문서가 저장되는 위치와 폴더 구조 정의
- [설계 문서 작성 순서](DESIGN_DOCUMENT_ORDER.md) - Glossary, 유스케이스, API 스펙의 작성 순서와 관계
- [유스케이스 표준](USE_CASE_STANDARDS.md) - 유스케이스 작성 표준
- [API 스펙 표준](API_SPEC_STANDARDS.md) - OpenAPI 스펙 작성 표준

### 개발 환경 관련
- [VS Code 개발 표준](VSCODE_DEVELOPMENT_GUIDELINES.md) - VS Code 워크스페이스 구성 및 설정 표준
- [Git 개발 가이드라인](GIT_DEVELOPMENT_GUIDELINES.md) - Git 브랜치 전략 및 커밋 규칙

## 템플릿
- [Glossary 템플릿](templates/glossary_template.md)
- [업무 유스케이스 템플릿](templates/business_use_case_template.md)
- [기능 유스케이스 템플릿](templates/functional_use_case_template.md)
- [OpenAPI 템플릿](templates/openapi_template.yaml)

## 빠른 시작

### 새 프로젝트 설정
1. 프로젝트 리포지토리에 dev-standards 추가:
   ```bash
   git submodule add https://github.com/threewindow-dev/dev-standards.git .dev-standards
   ```

2. 설계 문서 폴더 구조 생성:
   ```bash
   mkdir -p docs/design/use-cases/{business,functional}
   mkdir -p docs/design/api-specs
   ```

3. 템플릿을 사용하여 초기 문서 생성:
   ```bash
   cp .dev-standards/templates/glossary_template.md docs/design/glossary.md
   ```

상세한 내용은 각 표준 문서를 참조하세요.
