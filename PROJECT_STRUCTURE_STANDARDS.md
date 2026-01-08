# 프로젝트 구조 표준

## 목적
- 일관된 프로젝트 폴더 구조를 통해 코드 탐색성과 유지보수성을 향상시킵니다.
- 싱글 레포지토리와 멀티 레포지토리 프로젝트의 구조 표준을 제공합니다.

## 적용 대상
- threewindow-dev 조직의 모든 개발 프로젝트

## 싱글 레포지토리 프로젝트 구조

### 기본 원칙
- **서비스별 최상위 디렉토리**: 각 서비스(backend, frontend 등)가 독립적인 최상위 폴더를 가짐
- **서비스 내 완전한 구조**: 각 서비스 폴더 내에 `src/`, `tests/`, 설정 파일 등 모든 것을 포함
- **명확한 경계**: 서비스 간 의존성이 디렉토리 구조로 명확히 드러남

### 표준 구조

```
{project-name}-simple/
├── .dev-standards/           # dev-standards 서브모듈
├── .tool-versions           # 런타임 버전 정의
├── README.md
├── docs/                    # 프로젝트 설계 문서
│   └── design/
│       ├── glossary.md
│       ├── use-cases/
│       └── api-specs/
├── backend/                 # 백엔드 서비스 (또는 api, server 등)
│   ├── src/                 # 소스 코드
│   ├── tests/               # 테스트 코드
│   ├── .tool-versions       # 서비스별 런타임 버전 (선택사항)
│   ├── requirements.txt     # Python 의존성
│   ├── pyproject.toml       # Python 프로젝트 설정
│   └── README.md            # 서비스별 문서
├── frontend/                # 프론트엔드 서비스 (또는 web, client 등)
│   ├── src/                 # 소스 코드
│   ├── tests/               # 테스트 코드
│   ├── public/              # 정적 파일
│   ├── package.json         # Node.js 의존성
│   ├── tsconfig.json        # TypeScript 설정
│   └── README.md            # 서비스별 문서
├── shared/                  # 공통 코드 (선택사항)
│   ├── types/               # 공통 타입 정의
│   └── utils/               # 공통 유틸리티
└── scripts/                 # 빌드, 배포 등 스크립트 (선택사항)
```

### 서비스 이름 지정 규칙

**목적에 따른 명명:**
- `backend`, `frontend`: 기본적이고 명확한 이름
- `api`, `web`: 더 구체적인 역할 표현
- `server`, `client`: 클라이언트-서버 구조 강조
- `admin-panel`, `mobile-api`: 특정 기능 서비스

**권장 사항:**
- 프로젝트 규모가 작고 명확한 경우: `backend`, `frontend`
- 여러 API 서비스가 있는 경우: `user-api`, `payment-api`, `notification-api`
- 여러 클라이언트가 있는 경우: `web`, `mobile`, `admin`

### 왜 `/backend/src`가 `/src/backend`보다 나은가?

#### 1. 서비스 독립성
```
# ✅ 권장: 서비스별 독립적 구조
/backend/
  ├── src/
  ├── tests/
  ├── requirements.txt      # 독립적 의존성 관리
  └── pyproject.toml

# ❌ 비권장: 중앙화된 구조
/src/
  ├── backend/
  └── frontend/
/tests/
  ├── backend/
  └── frontend/
/requirements.txt           # 의존성 혼재
```

#### 2. 확장성
```
# ✅ 새 서비스 추가 용이
/backend/
/frontend/
/admin-panel/               # 동일한 패턴으로 추가
/mobile-api/
```

#### 3. 멀티레포 전환 용이
각 서비스 폴더가 완전한 구조를 가지므로, 필요시 별도 리포지토리로 쉽게 분리 가능:
```bash
# backend 서비스를 별도 리포지토리로 분리
git subtree split -P backend -b backend-branch
```

#### 4. 모노레포 도구 호환성
Nx, Turborepo, Lerna 등 주요 모노레포 도구들이 이 구조를 표준으로 사용:
```
/apps/
  /backend/
  /frontend/
/packages/
  /shared/
```

#### 5. 빌드 및 배포 독립성
- 각 서비스가 독립적으로 빌드, 테스트, 배포 가능
- CI/CD 파이프라인에서 변경된 서비스만 빌드 가능
- Docker 이미지 생성 시 서비스별 독립적 관리

## 멀티 레포지토리 프로젝트 구조

### 기본 원칙
- **각 서비스는 독립된 리포지토리**: 완전히 분리된 개발, 배포 라이프사이클
- **공통 코드는 별도 패키지**: npm 패키지, Python 패키지 등으로 공유

### 표준 구조

```
{project-name}/                    # 루트 폴더 (워크스페이스)
├── gateway-service/               # 각 서비스는 독립 리포지토리
│   ├── .git/
│   ├── .tool-versions
│   ├── src/
│   ├── tests/
│   └── README.md
├── auth-service/
│   ├── .git/
│   ├── .tool-versions
│   ├── src/
│   ├── tests/
│   └── README.md
├── payment-service/
│   ├── .git/
│   └── ...
└── web-client/
    ├── .git/
    └── ...
```

## .tool-versions 파일 위치

### 싱글 레포지토리
**옵션 1: 프로젝트 루트에만 작성 (권장)**
- 모든 서비스가 동일한 런타임 버전 사용
- 관리가 간단하고 일관성 유지 용이

```
{project-name}-simple/
├── .tool-versions          # 전체 프로젝트 런타임 버전
├── backend/
└── frontend/
```

**옵션 2: 서비스별 작성 (고급)**
- 서비스마다 다른 런타임 버전 필요 시
- 예: backend는 Python 3.13, 데이터 분석 서비스는 Python 3.12

```
{project-name}-simple/
├── .tool-versions          # 기본 버전
├── backend/
│   └── .tool-versions      # backend 전용 버전 (선택사항)
└── frontend/
    └── .tool-versions      # frontend 전용 버전 (선택사항)
```

### 멀티 레포지토리
- 각 리포지토리 루트에 `.tool-versions` 작성
- 서비스별 완전히 독립적인 버전 관리

## 공통 코드 관리

### 싱글 레포지토리
`/shared/` 디렉토리 사용:
```
/shared/
  ├── types/                # TypeScript 타입 정의
  │   ├── user.ts
  │   └── api.ts
  ├── constants/            # 공통 상수
  └── utils/                # 공통 유틸리티
```

### 멀티 레포지토리
별도 패키지로 관리:
```
# npm 패키지
@{org}/{project-name}-shared

# Python 패키지
{project-name}-shared
```

## 배포 설정 관리

### 기본 원칙
- **배포 설정은 소스 코드**: Git으로 버전 관리되어야 함
- **빌드 결과물 폴더에 두지 않기**: `/dist`, `/build` 등은 `.gitignore`에 포함되므로 부적절
- **명확한 위치**: 배포 관련 파일들을 한 곳에 모아 관리

### ❌ 잘못된 위치

```
{project-name}-simple/
├── backend/
│   └── dist/
│       └── Dockerfile      # ❌ 빌드 결과물 폴더
└── frontend/
    └── dist/
        └── deployment.yaml # ❌ Git에서 무시될 수 있음
```

**문제점:**
1. `/dist`, `/build`는 빌드 결과물을 위한 임시 폴더
2. 일반적으로 `.gitignore`에 포함되어 버전 관리되지 않음
3. 빌드할 때마다 삭제/재생성될 수 있음
4. 배포 설정은 소스 코드처럼 버전 관리가 필요함

### ✅ 권장 위치

#### 옵션 1: 프로젝트 루트 (간단한 프로젝트)

```
{project-name}-simple/
├── Dockerfile                    # 단일 서비스 또는 공통 Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .github/                      # GitHub Actions CI/CD
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── backend/
└── frontend/
```

**적합한 경우:**
- 간단한 배포 설정 (1~2개 파일)
- 단일 컨테이너 배포
- 배포 파이프라인이 단순함

#### 옵션 2: 서비스별 배포 설정

```
{project-name}-simple/
├── backend/
│   ├── Dockerfile               # backend 전용
│   ├── .dockerignore
│   └── src/
├── frontend/
│   ├── Dockerfile               # frontend 전용
│   ├── .dockerignore
│   └── src/
└── docker-compose.yml           # 루트에서 통합 관리
```

**적합한 경우:**
- 각 서비스가 독립적으로 배포됨
- 서비스별 빌드 설정이 다름
- 마이크로서비스 아키텍처

#### 옵션 3: deployment/ 디렉토리 (복잡한 프로젝트)

```
{project-name}-simple/
├── deployment/                   # 또는 deploy/, infra/, .deploy/
│   ├── docker/
│   │   ├── backend.Dockerfile
│   │   ├── frontend.Dockerfile
│   │   └── docker-compose.yml
│   ├── kubernetes/
│   │   ├── backend/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── configmap.yaml
│   │   ├── frontend/
│   │   │   └── ...
│   │   └── ingress.yaml
│   ├── terraform/               # 인프라 코드
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── ansible/                 # 설정 관리
│   │   └── playbook.yml
│   └── scripts/                 # 배포 스크립트
│       ├── deploy.sh
│       └── rollback.sh
├── .github/
│   └── workflows/
│       └── deploy.yml
├── backend/
└── frontend/
```

**적합한 경우:**
- 복잡한 인프라 설정 (Kubernetes, Terraform 등)
- 여러 환경 관리 (dev, staging, production)
- IaC (Infrastructure as Code) 사용

### 배포 관련 파일 종류

#### 1. 컨테이너 설정
```
Dockerfile
docker-compose.yml
docker-compose.prod.yml
.dockerignore
```

#### 2. 오케스트레이션
```
# Kubernetes
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
ingress.yaml

# Docker Swarm
stack.yml
```

#### 3. 인프라 코드
```
# Terraform
*.tf

# AWS CloudFormation
*.yaml, *.json

# Ansible
playbook.yml
```

#### 4. CI/CD 설정
```
# GitHub Actions
.github/workflows/*.yml

# GitLab CI
.gitlab-ci.yml

# Jenkins
Jenkinsfile

# CircleCI
.circleci/config.yml
```

#### 5. 환경 설정
```
.env.example              # 환경 변수 템플릿 (버전 관리 O)
.env                      # 실제 환경 변수 (버전 관리 X, .gitignore)
.env.development
.env.production
```

### 권장 디렉토리 이름

**일반적인 관행:**
- `deployment/`: 가장 명확하고 직관적
- `deploy/`: 간결한 버전
- `infra/`: 인프라 관련 강조
- `.deploy/`: 숨김 폴더로 관리 (덜 일반적)

**피해야 할 이름:**
- `dist/`: 빌드 결과물과 혼동
- `build/`: 빌드 결과물과 혼동
- `config/`: 애플리케이션 설정과 혼동

### 환경별 설정 관리

```
deployment/
├── docker/
│   ├── Dockerfile.backend
│   ├── Dockerfile.frontend
│   ├── docker-compose.yml          # 개발 환경
│   ├── docker-compose.prod.yml     # 프로덕션 환경
│   └── docker-compose.staging.yml  # 스테이징 환경
└── kubernetes/
    ├── base/                        # 공통 설정
    │   └── ...
    └── overlays/                    # 환경별 override
        ├── development/
        ├── staging/
        └── production/
```

### .gitignore 설정 주의사항

**버전 관리 해야 할 것:**
```gitignore
# 배포 설정 파일은 포함 (기본적으로 추적됨)
Dockerfile
docker-compose*.yml
deployment/
.github/

# 환경 변수 템플릿은 포함
.env.example
```

**버전 관리 하지 말아야 할 것:**
```gitignore
# 빌드 결과물
dist/
build/
*.pyc
node_modules/

# 실제 환경 변수 (민감 정보 포함)
.env
.env.local
.env.*.local

# 비밀키, 인증서
*.key
*.pem
secrets/
```

### 예제: 권장 전체 구조

```
{project-name}-simple/
├── .dev-standards/
├── .tool-versions
├── .gitignore
├── README.md
├── docs/
├── deployment/                      # 배포 설정 중앙 관리
│   ├── docker/
│   │   ├── backend.Dockerfile
│   │   ├── frontend.Dockerfile
│   │   ├── docker-compose.yml
│   │   └── docker-compose.prod.yml
│   ├── kubernetes/
│   │   ├── backend/
│   │   └── frontend/
│   └── scripts/
│       └── deploy.sh
├── .github/                         # CI/CD
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── backend/
│   ├── src/
│   ├── tests/
│   ├── dist/                       # ❌ 배포 설정 여기 두지 않음
│   └── README.md
├── frontend/
│   ├── src/
│   ├── tests/
│   ├── dist/                       # ❌ 배포 설정 여기 두지 않음
│   └── README.md
└── shared/
```

## 예제

### 예제 1: FastAPI + React 풀스택 프로젝트

```
fastexit-simple/
├── .dev-standards/
├── .tool-versions
├── README.md
├── docs/
│   └── design/
├── api/                     # FastAPI 백엔드
│   ├── src/
│   │   ├── main.py
│   │   ├── routers/
│   │   ├── models/
│   │   └── services/
│   ├── tests/
│   ├── requirements.txt
│   └── README.md
├── web/                     # React 프론트엔드
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   └── pages/
│   ├── tests/
│   ├── package.json
│   └── README.md
└── shared/
    └── types/
        └── api-types.ts
```

### 예제 2: Spring Boot + React 프로젝트

```
ecommerce-simple/
├── .dev-standards/
├── .tool-versions
├── README.md
├── docs/
├── server/                  # Spring Boot 백엔드
│   ├── src/
│   │   ├── main/
│   │   │   └── java/
│   │   └── test/
│   ├── build.gradle
│   └── README.md
├── client/                  # React 프론트엔드
│   ├── src/
│   ├── tests/
│   ├── package.json
│   └── README.md
└── shared/
    └── types/
```

### 예제 3: 멀티 서비스 싱글 레포지토리

```
platform-simple/
├── .tool-versions
├── user-api/                # 사용자 관리 API
│   ├── src/
│   └── tests/
├── payment-api/             # 결제 API
│   ├── src/
│   └── tests/
├── notification-api/        # 알림 API
│   ├── src/
│   └── tests/
├── web/                     # 웹 클라이언트
│   ├── src/
│   └── tests/
├── admin/                   # 관리자 패널
│   ├── src/
│   └── tests/
└── shared/
    ├── types/
    └── utils/
```

## 체크리스트

### 싱글 레포지토리 프로젝트 생성 시
- [ ] 서비스별 최상위 디렉토리 생성 (backend, frontend 등)
- [ ] 각 서비스 내 `src/`, `tests/` 디렉토리 생성
- [ ] 프로젝트 루트에 `.tool-versions` 작성
- [ ] 각 서비스에 `README.md` 작성
- [ ] `docs/design/` 폴더 구조 생성
- [ ] `.dev-standards` 서브모듈 추가
- [ ] 공통 코드 필요 시 `shared/` 디렉토리 생성
- [ ] 배포 설정 위치 결정 (루트, 서비스별, 또는 `deployment/`)
- [ ] `.gitignore`에 빌드 결과물 추가 (`dist/`, `build/` 등)
- [ ] `.env.example` 작성 (환경 변수 템플릿)

### 멀티 레포지토리 프로젝트 생성 시
- [ ] 각 서비스별 독립 리포지토리 생성
- [ ] 각 리포지토리에 `.tool-versions` 작성
- [ ] 각 리포지토리에 `.dev-standards` 서브모듈 추가
- [ ] 공통 코드는 별도 패키지로 관리
- [ ] 각 리포지토리에 배포 설정 추가

## 참고 자료
- [VS Code 개발 표준](VSCODE_DEVELOPMENT_GUIDELINES.md) - 워크스페이스 구성
- [런타임 버전 선택 표준](RUNTIME_VERSION_STANDARDS.md) - 버전 관리
- [Nx Monorepo Best Practices](https://nx.dev/concepts/more-concepts/applications-and-libraries)
- [Turborepo Structure](https://turbo.build/repo/docs/handbook/what-is-a-monorepo)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
