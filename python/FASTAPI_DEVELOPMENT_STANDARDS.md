# Python FastAPI 개발 표준 — DDD / Clean Architecture (Python 3.13)

문서 탐색: [네이밍 인덱스](NAMING_CONVENTION.md) · [데이터베이스 계층](DATABASE_STANDARDS.md) · [에러 처리](ERROR_HANDLING.md) · [테스트 전략](TESTING_STRATEGY.md)

## 개요
- 도메인 주도 설계(DDD)와 Clean Architecture 원칙을 적용하여 유지보수성과 확장성을 극대화합니다.
- 4 레이어를 도메인 중심으로 재구성: interface, application, domain, infra.
- 모든 예시는 Python 3.13 문법을 사용합니다: list[T], dict[str, Any], T | None, @dataclass(slots=True).

## 레이어 매핑
| DDD 레이어 | 표준 레이어 매핑 | 주요 책임 |
|------------|------------------|-----------|
| interface  | routers, schemas | 외부 API 경계, 요청/응답 검증, 클라이언트별 라우팅 |
| application| services (+ dtos)| 유스케이스(AppService, IntService), 트랜잭션/권한, DTO 변환 |
| domain     | models, domain services, protocols | 도메인 모델/VO, 도메인 규칙, Repository/Client Protocol 정의 |
| infra      | repositories, clients, database | DB/외부 시스템 구현체, Repository/Client 구현, 연결 관리 |

## 폴더 구조: 도메인 모듈 기준 (예시)
```
backend/
└── src/
    ├── main.py                   # 진입점
    ├── dependencies.py           # 프로젝트 DI 설정 (subdomains 참조)
    ├── core/                     # 프로젝트별 설정 (재사용 불가)
    │   ├── exception_handlers.py
    │   ├── logging.py
    │   ├── common_responses.py   # 프로젝트 OpenAPI 응답 정의
    │   ├── protocols/            # 프로젝트 인터페이스
    │   └── infra/                # 프로젝트 구현체
    ├── shared/                   # 재사용 가능 공통 코드
    │   ├── errors.py             # 기본 예외 계층
    │   ├── schemas.py            # ApiResponse<T>
    │   ├── protocols/            # 범용 추상 인터페이스
    │   │   ├── database.py       # DatabasePool Protocol
    │   │   └── transaction.py    # Transaction Protocol
    │   └── infra/
    │       └── database.py       # SQLAlchemy/Psycopg 구현
    └── subdomains/                  # 도메인 모듈 집합 
        ├── user/
        │   ├── interface/
        │   │   ├── routers/
        │   │   │   ├── __init__.py      # 서브도메인 router 통합 (prefix 책임)
        │   │   │   └── user_router.py   # 개별 라우터 (경로만 정의)
        │   │   └── schemas/
        │   ├── application/
        │   │   ├── dtos/
        │   │   └── services/    # AppService, IntService
        │   ├── domain/
        │   │   ├── models/      # Domain Model, VO
        │   │   ├── services/    # DomainService
        │   │   └── protocols/   # Repository/Client Protocol(인터페이스)
        │   └── infra/
        │       ├── repositories/
        │       └── clients/
        └── employee/
            └── ... (동일 패턴)
```

## 폴더 구조: 도메인 모듈 구분 없음
```
backend/
└── src/
    ├── interface/     # routers, schemas
    ├── application/   # dtos, services(AppService, IntService)
    ├── domain/        # models, services, protocols
    └── infra/         # repositories, clients, database
```

## Router 구성 규칙

### 서브도메인별 Router 통합

**개별 Router 파일** (`user_router.py`):
- prefix 없이 경로만 정의
- tags와 responses만 설정
- 예: `router = APIRouter(tags=["users"])`

**통합 Router** (`routers/__init__.py`):
- 모든 개별 router를 통합하고 prefix 설정
- 서브도메인이 API 경로 책임
- main.py에서는 이 router를 import

```python
# subdomains/user/interface/routers/user_router.py
from fastapi import APIRouter

router = APIRouter(tags=["users"])

@router.post("/")
async def create_user(...):
    ...

@router.get("/{user_id}")
async def get_user(...):
    ...

# subdomains/user/interface/routers/__init__.py
from fastapi import APIRouter
from .user_router import router as user_router

router = APIRouter(prefix="/api/users")
router.include_router(user_router)

__all__ = ["router"]

# main.py
from subdomains.user.interface.routers import router as user_router
app.include_router(user_router)
```

## 서비스 레이어 구분
- AppService: 트랜잭션 경계, 권한 체크, 유스케이스 조합, 도메인 Protocol만 의존 (application/services)
- IntService: 내부/외부 시스템 간 연동(필요 시), 메시지 브로커/3rd-party 호출 (application/services)
- DomainService: 엔티티 외부의 복잡한 도메인 규칙 구현(단순 Repository 위임 금지) (domain/services)

## DTO vs Schema 분리
- Schema (interface/schemas): HTTP 경계에서의 요청/응답 검증 및 문서화용
- DTO (application/dtos): 계층 간 데이터 전달 전용, 서비스/도메인 경계 유지
- 매핑: Router에서 Schema → DTO, Service 결과를 DTO → Schema

## 의존성 역전(Dependency Inversion)
- application은 domain의 Protocol(인터페이스)만 의존
- infra 구현체는 domain의 Protocol을 구현
- 실제 바인딩은 src/dependencies.py에서 의존성 주입으로 연결

## HTTP 상태 코드 및 응답 규칙
- **200 OK**: 성공 응답 (GET, PATCH, DELETE 등) - 본문 포함
- **201 Created**: 리소스 생성 성공 (POST) - 생성된 리소스 본문 포함
- **204 No Content**: 성공하지만 응답 본문 없음 (사용 권장하지 않음, 표준은 200 권장)
- **400 Bad Request**: 비즈니스 규칙 위반, 입력값 검증 실패
- **401 Unauthorized**: 인증 실패
- **403 Forbidden**: 권한 없음
- **404 Not Found**: 리소스 없음
- **500 Internal Server Error**: 예상치 못한 서버 오류

삭제(DELETE) 작업은 **HTTP 200 OK (본문 포함)**를 권장합니다:
- 클라이언트가 삭제된 리소스 정보를 확인 가능
- 응답: `{code: "DELETED", message: "...", data: {id, deleted_at}}`


## 예시 (Python 3.13)
```
# domain/models/user.py
from dataclasses import dataclass, field
from datetime import datetime

@dataclass(slots=True)
class User:
    id: int | None = None
    email: str
    name: str
    created_at: datetime = field(default_factory=datetime.utcnow)

# domain/protocols/user_repository_protocol.py
from typing import Protocol
from .user import User

class UserRepositoryProtocol(Protocol):
    def find_by_id(self, id: int) -> User | None: ...
    def save(self, user: User) -> User: ...

# application/services/user_app_service.py
from ..domain.protocols.user_repository_protocol import UserRepositoryProtocol
from ..domain.models.user import User

class UserAppService:
    def __init__(self, repo: UserRepositoryProtocol):
        self.repo = repo

    def create_user(self, email: str, name: str) -> User:
        user = User(email=email, name=name)
        return self.repo.save(user)

# infra/repositories/user_repository.py
from ..domain.protocols.user_repository_protocol import UserRepositoryProtocol
from ..domain.models.user import User

class UserRepository(UserRepositoryProtocol):
    def find_by_id(self, id: int) -> User | None:
        ...  # 실제 DB 접근 구현 (예: SELECT)

    def save(self, user: User) -> User:
        ...  # 실제 DB 접근 구현 (예: INSERT/UPDATE)

# dependencies.py (src/)
from application.services.user_app_service import UserAppService
from infra.repositories.user_repository import UserRepository

def get_user_app_service() -> UserAppService:
    return UserAppService(UserRepository())
```

## 패키지 노출 규칙 (__init__.py)
- 하위 패키지는 공개 클래스/함수를 명시적으로 export (from .file import Class)
- 새 공개 심볼 추가 시 __init__.py 동시 업데이트 필수

## 계층별 의존성 규칙 (요약)
- common/domain: 의존성 없음
- common/infra: common/domain만 의존 가능
- domain/models: 의존성 없음
- domain/services: domain/models 의존 가능
- application/dtos: 의존성 없음
- application/services: application/dtos, domain/models, domain/services 의존 가능
- infra/repositories, infra/clients: domain/models 의존 가능
- interface/schemas: 의존성 없음
- interface/routers: interface/schemas, application/dtos, application/services 의존 가능

## 의존성 주입 및 트랜잭션
- application은 infra 직접 의존 금지, Protocol만 의존
- 트랜잭션은 AppService에서만 주입/관리
- Repository 생성자에서 컨텍스트 매니저 직접 주입 금지

## 모듈 간 의존성 (예시)
- common: 모든 도메인에서 참조 가능
- user: common만 참조, 다른 도메인 직접 참조 금지
- order: common, user만 참조 가능
- account/authz: 직접 참조 금지. common/domain의 Protocol 참조 후 구현체를 주입

## Router 클라이언트 분리
- interface/routers/client/{manager,user,device,...} 로 클라이언트 유형별 라우팅 분리
- 공통 라우팅/권한 정책은 AppService/IntService에서 관리

## 마이그레이션 팁 (단일 파일 → DDD)
1) schemas와 routers를 interface로 이동
2) 요청/응답 Schema와 내부 전달 DTO 분리
3) 도메인 모델/서비스/Protocol 정의 후, 기존 Repository를 Protocol 구현으로 변환
4) AppService로 유스케이스/트랜잭션/권한 로직 집약
5) dependencies.py (src/)에서 의존성 주입 구성

## 버전 이력
- 1.2.3 (2026-01-11): 예시 도메인을 Company에서 User로 변경
- 1.2.2 (2026-01-10): 문서 정리 — DDD-only로 재작성, 중복/레거시 섹션 제거
- 1.2.1 (2026-01-10): 모든 예제를 Python 3.13 타입 힌팅으로 업데이트
- 1.2.0 (2026-01-10): 파일을 DDD 섹션만 남기도록 정리
- 1.1.0 (2026-01-10): DDD/Clean Architecture 확장 섹션 추가
- 1.0.0 (2026-01-10): 초기 버전 작성
