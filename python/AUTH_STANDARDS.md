# FastAPI 인증 표준 (AUTH_STANDARDS)

## 목차
1. [개요](#개요)
2. [아키텍처](#아키텍처)
3. [핵심 구성 요소](#핵심-구성-요소)
4. [구현 가이드](#구현-가이드)
5. [Swagger UI 통합](#swagger-ui-통합)
6. [사용 예제](#사용-예제)
7. [감사로그 및 Command 통합](#감사로그-및-command-통합)

---

## 개요

본 문서는 FastAPI 기반 백엔드 서버에서 인증(Authentication)을 처리하기 위한 표준을 정의합니다.

### 주요 특징
- **Bearer 토큰 기반**: 클라이언트는 HTTP Authorization 헤더를 통해 Bearer 토큰 전달
- **ContextVar 활용**: 요청 생명주기 동안 인증 정보를 스레드-안전하게 관리
- **선택적 Role**: 프로젝트 요구사항에 따라 role 정보 포함 가능
- **데코레이터 기반 접근 제어**: `@authenticate_required` 데코레이터로 보호된 엔드포인트 구현

---

## 아키텍처

### 흐름도

```
HTTP Request
    ↓
Authorization 헤더 파싱 (Bearer Token)
    ↓
토큰 검증 (토큰 디코딩, 만료 확인 등)
    ↓
AuthenticatedUser 생성
    ↓
ContextVar에 저장
    ↓
핸들러 실행 (@authenticate_required 데코레이터 체크)
    ↓
감사로그 기록 / Command 객체 생성 시 사용자 정보 추가
    ↓
응답 반환
```

### 계층 구조

- **Middleware**: 모든 요청에서 토큰을 검증하고 ContextVar에 저장
- **Decorator**: 특정 엔드포인트의 인증 요구사항 명시
- **Models**: 인증 정보 구조 정의 (`AuthenticatedUser`)
- **Dependencies**: FastAPI 의존성 주입을 통한 인증된 사용자 접근

---

## 핵심 구성 요소

### 계층 구조

```
Application Layer (core/)
├── middleware/           → shared.protocols.auth 참조 (Protocol)
└── ...

Infrastructure Layer (shared/)
├── context/              ← 인증 정보 관리
├── protocols/
│   └── auth.py          ← TokenManager Protocol 정의
└── infra/
    └── auth.py          ← JWTTokenManager 구현
```

### 의존성 방향 (DIP - Dependency Inversion Principle)

```
core/middleware/AuthenticationMiddleware
    ↓
shared.protocols.auth.TokenManager (Protocol)
    ↑
shared.infra.auth.JWTTokenManager (구현체)
```

**핵심 원칙:**
- Middleware는 **구체적 구현**이 아닌 **Protocol에 의존**
- 구현체는 Protocol을 **구현**
- 이를 통해 TokenManager 구현을 자유롭게 변경 가능 (JWT, OAuth 등)

### 1. AuthenticatedUser 모델

```python
# shared/context/auth_context.py

from contextvars import ContextVar
from dataclasses import dataclass

@dataclass
class AuthenticatedUser:
    """
    인증된 사용자 정보
    
    Attributes:
        user_id: 사용자 고유 ID (필수)
        role: 사용자의 역할 (선택적, 프로젝트별로 다를 수 있음)
        metadata: 추가 정보 저장 (선택적)
    """
    user_id: str
    role: str | None = None
    metadata: dict | None = None
```

### 2. ContextVar를 이용한 전역 상태 관리

#### 계층 구조
```
shared/
└── context/                     # 인프라 레이어 (기술적 구현)
    ├── __init__.py              # Facade 패턴으로 통합 인터페이스 제공
    ├── auth_context.py          # 인증 컨텍스트
    └── transaction_context.py   # 트랜잭션 컨텍스트 (기존)
```

#### Auth Context 구현
```python
# shared/context/auth_context.py

from contextvars import ContextVar
from dataclasses import dataclass

@dataclass
class AuthenticatedUser:
    """
    인증된 사용자 정보
    
    Attributes:
        user_id: 사용자 고유 ID (필수)
        role: 사용자의 역할 (선택적, 프로젝트별로 다를 수 있음)
        metadata: 추가 정보 저장 (선택적)
    """
    user_id: str
    role: str | None = None
    metadata: dict | None = None

# Transaction Context와 함께 관리되는 ContextVar
authenticated_user_context: ContextVar[AuthenticatedUser | None] = ContextVar(
    'authenticated_user',
    default=None
)

def get_authenticated_user() -> AuthenticatedUser | None:
    """현재 요청의 인증된 사용자 정보 조회"""
    return authenticated_user_context.get()

def set_authenticated_user(user: AuthenticatedUser) -> None:
    """현재 요청의 인증된 사용자 정보 설정"""
    authenticated_user_context.set(user)

def clear_authenticated_user() -> None:
    """현재 요청의 인증된 사용자 정보 초기화"""
    authenticated_user_context.set(None)
```

#### Facade 패턴 (통합 인터페이스)
```python
# shared/context/__init__.py

"""
컨텍스트 관리 통합 모듈 (Facade Pattern)

모든 ContextVar 관련 기능을 한 곳에서 import 가능하도록 Facade 제공.
Transaction Context와 Auth Context를 통합 관리합니다.
"""

# Auth Context
from .auth_context import (
    AuthenticatedUser,
    get_authenticated_user,
    set_authenticated_user,
    clear_authenticated_user,
)

# Transaction Context
from .transaction_context import (
    get_transaction,
    set_transaction,
    clear_transaction,
)

def clear_all_contexts() -> None:
    """모든 요청 컨텍스트 초기화"""
    clear_authenticated_user()
    clear_transaction()

__all__ = [
    "AuthenticatedUser",
    "get_authenticated_user",
    "set_authenticated_user",
    "clear_authenticated_user",
    "get_transaction",
    "set_transaction",
    "clear_transaction",
    "clear_all_contexts",
]
```
"""
컨텍스트 관리 통합 모듈

모든 ContextVar 관련 기능을 한 곳에서 import 가능하도록 Facade 제공
"""

# Auth Context
from .auth_context import (
    authenticated_user_context,
    get_authenticated_user,
    set_authenticated_user,
    clear_authenticated_user,
)

# Transaction Context
from .transaction_context import (
    transaction_context,
    get_transaction,
    set_transaction,
    clear_transaction,
)

# 전체 초기화 유틸리티
def clear_all_contexts() -> None:
    """모든 요청 컨텍스트 초기화"""
    clear_authenticated_user()
    clear_transaction()

__all__ = [
    # Auth
    "authenticated_user_context",
    "get_authenticated_user",
    "set_authenticated_user",
    "clear_authenticated_user",
    # Transaction
    "transaction_context",
    "get_transaction",
    "set_transaction",
    "clear_transaction",
    # Utilities
    "clear_all_contexts",
]
```

### 3. TokenManager Protocol

#### TokenManager 인터페이스 정의
```python
# shared/protocols/auth.py

from typing import Protocol, dict

class TokenManager(Protocol):
    """
    JWT 토큰 생성 및 검증을 담당하는 Protocol
    
    구현체: shared.infra.auth.JWTTokenManager
    
    의존성 역전 원칙(DIP)에 따라 Middleware는 
    구체적 구현체가 아닌 Protocol에 의존합니다.
    """
    
    def verify_token(self, token: str) -> dict | None:
        """
        토큰 검증 및 payload 추출
        
        Args:
            token: JWT 토큰 문자열
            
        Returns:
            성공 시 payload dict, 실패 시 None
        """
        ...
    
    def create_token(self, user_id: str, role: str | None = None, 
                    expires_in_minutes: int = 60) -> str:
        """
        JWT 토큰 생성
        
        Args:
            user_id: 사용자 ID
            role: 사용자 역할 (선택적)
            expires_in_minutes: 만료 시간 (분 단위)
            
        Returns:
            JWT 토큰 문자열
        """
        ...
```

#### TokenManager 구현체
```python
# shared/infra/auth.py

import jwt
from datetime import datetime, timedelta

class JWTTokenManager:
    """JWT 토큰 생성 및 검증 구현체
    
    Protocol: shared.protocols.auth.TokenManager
    """
    
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        """
        Args:
            secret_key: JWT 서명에 사용할 비밀 키
            algorithm: JWT 알고리즘 (기본값: HS256)
        """
        self.secret_key = secret_key
        self.algorithm = algorithm
    
    def verify_token(self, token: str) -> dict | None:
        """
        토큰 검증 및 payload 추출
        
        Args:
            token: JWT 토큰 문자열
            
        Returns:
            성공 시 payload dict, 실패 시 None
        """
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.ExpiredSignatureError:
            # 토큰 만료
            return None
        except jwt.InvalidTokenError:
            # 유효하지 않은 토큰
            return None
    
    def create_token(self, user_id: str, role: str | None = None, 
                    expires_in_minutes: int = 60) -> str:
        """
        JWT 토큰 생성
        
        Args:
            user_id: 사용자 ID
            role: 사용자 역할 (선택적)
            expires_in_minutes: 만료 시간 (분 단위)
            
        Returns:
            JWT 토큰 문자열
        """
        payload = {
            'user_id': user_id,
            'exp': datetime.utcnow() + timedelta(minutes=expires_in_minutes)
        }
        if role:
            payload['role'] = role
        
        token = jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
        return token
```

### 4. 인증 Middleware

```python
# core/middleware.py

from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse
import logging
from typing import Optional
from shared.context import set_authenticated_user, clear_authenticated_user
from shared.protocols.auth import TokenManager  # Protocol 의존

logger = logging.getLogger(__name__)

class AuthenticationMiddleware(BaseHTTPMiddleware):
    """
    모든 요청에서 Authorization 헤더를 검증하고
    ContextVar에 인증 정보를 저장하는 Middleware
    
    TokenManager Protocol에 의존하여 구체적 구현과 분리됩니다.
    Transaction Context와 함께 요청 생명주기 동안 인증 정보를 관리합니다.
    """
    
    def __init__(self, app, token_manager: TokenManager):
        super().__init__(app)
        self.token_manager = token_manager
    
    async def dispatch(self, request: Request, call_next):
        # Authorization 헤더 추출
        auth_header = request.headers.get('Authorization', '')
        
        # Bearer 토큰 추출
        token = self._extract_bearer_token(auth_header)
        
        if token:
            # 토큰 검증
            payload = self.token_manager.verify_token(token)
            if payload:
                # AuthenticatedUser 생성 및 ContextVar에 저장
                authenticated_user = AuthenticatedUser(
                    user_id=payload.get('user_id'),
                    role=payload.get('role'),
                    metadata=payload.get('metadata')
                )
                set_authenticated_user(authenticated_user)
                logger.debug(f"User authenticated: {authenticated_user.user_id}")
        
        # 요청 처리
        response = await call_next(request)
        
        # 응답 후 ContextVar 초기화 (Transaction Context와 함께 초기화됨)
        clear_authenticated_user()
        
        return response
    
    @staticmethod
    def _extract_bearer_token(auth_header: str) -> str | None:
        """Authorization 헤더에서 Bearer 토큰 추출"""
        if not auth_header.startswith('Bearer '):
            return None
        return auth_header[7:]  # 'Bearer ' 이후의 문자열
```

### 5. 인증 필수 데코레이터

```python
# shared/decorators.py

from functools import wraps
from fastapi import HTTPException, status
from shared.context import get_authenticated_user

def authenticate_required(func):
    """
    인증된 사용자만 접근 가능한 엔드포인트를 표시하는 데코레이터
    
    Usage:
        @router.get("/protected")
        @authenticate_required
        async def protected_endpoint():
            user = get_authenticated_user()
            ...
    """
    @wraps(func)
    async def wrapper(*args, **kwargs):
        authenticated_user = get_authenticated_user()
        
        if not authenticated_user:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="인증이 필요합니다",
                headers={"WWW-Authenticate": "Bearer"},
            )
        
        return await func(*args, **kwargs)
    
    return wrapper

def role_required(*allowed_roles: str):
    """
    특정 역할을 가진 사용자만 접근 가능한 엔드포인트를 표시하는 데코레이터
    
    Usage:
        @router.delete("/admin-only")
        @role_required("admin", "superuser")
        async def admin_endpoint():
            user = get_authenticated_user()
            ...
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            authenticated_user = get_authenticated_user()
            
            if not authenticated_user:
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="인증이 필요합니다",
                    headers={"WWW-Authenticate": "Bearer"},
                )
            
            if authenticated_user.role not in allowed_roles:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail="이 작업을 수행할 권한이 없습니다"
                )
            
            return await func(*args, **kwargs)
        
        return wrapper
    return decorator
```

---

## 구현 가이드

### Step 1: Middleware 등록

```python
# main.py

from fastapi import FastAPI
from core.middleware import AuthenticationMiddleware
from shared.infra.auth import JWTTokenManager  # 구현체 import
from config import settings

app = FastAPI()

# TokenManager 초기화 (구현체)
token_manager = JWTTokenManager(
    secret_key=settings.SECRET_KEY,
    algorithm="HS256"
)

# Middleware 등록 (다른 middleware 전에 등록)
app.add_middleware(AuthenticationMiddleware, token_manager=token_manager)

# 라우터 등록
from subdomains.user.interface.routers import user_router
app.include_router(user_router)
```

### Step 2: 라우터에 데코레이터 적용

```python
# subdomains/user/interface/routers/user.py

from fastapi import APIRouter
from shared.decorators import authenticate_required, role_required
from shared.context import get_authenticated_user

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/me")
@authenticate_required
async def get_current_user():
    """현재 인증된 사용자 정보 조회"""
    user = get_authenticated_user()
    return {
        "user_id": user.user_id,
        "role": user.role
    }

@router.delete("/users/{user_id}")
@role_required("admin")
async def delete_user(user_id: str):
    """admin만 사용자 삭제 가능"""
    current_user = get_authenticated_user()
    # ... 사용자 삭제 로직
    return {"deleted": True}
```

---

## Swagger UI 통합

FastAPI의 Swagger UI에서 인증 토큰을 직접 입력하여 API를 테스트할 수 있도록 OAuth2PasswordBearer를 통합합니다.

### 개요

- **OAuth2PasswordBearer**: FastAPI의 보안 유틸리티로, Swagger UI에 "Authorize" 버튼을 자동으로 추가합니다.
- **자동 Security 추가**: `@authenticate_required` 데코레이터가 적용된 엔드포인트에 자동으로 security 정보를 추가합니다.
- **환경 변수 설정**: `KEYCLOAK_TOKEN_URL` 환경 변수를 통해 토큰 발급 URL을 설정합니다.

### 설정 방법

#### 1. 환경 변수 설정

```bash
# .env 파일
KEYCLOAK_TOKEN_URL=https://keycloak.example.com/realms/your-realm/protocol/openid-connect/token
```

#### 2. OAuth2PasswordBearer 초기화

```python
# dependencies.py

import os
from fastapi.security import OAuth2PasswordBearer

# 전역 OAuth2PasswordBearer 인스턴스
_oauth2_scheme: OAuth2PasswordBearer | None = None

def get_oauth2_scheme() -> OAuth2PasswordBearer:
    """OAuth2PasswordBearer 스킴 생성 및 반환

    Returns:
        OAuth2PasswordBearer: OAuth2 스킴 인스턴스

    Raises:
        RuntimeError: KEYCLOAK_TOKEN_URL 환경 변수가 설정되지 않은 경우
    """
    global _oauth2_scheme

    if _oauth2_scheme is None:
        token_url = os.getenv("KEYCLOAK_TOKEN_URL")

        if not token_url:
            raise RuntimeError(
                "KEYCLOAK_TOKEN_URL environment variable is required for OAuth2 scheme"
            )

        _oauth2_scheme = OAuth2PasswordBearer(tokenUrl=token_url, auto_error=False)

    return _oauth2_scheme
```

#### 3. Application Lifespan에서 초기화

```python
# main.py

from dependencies import get_oauth2_scheme
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    """애플리케이션 생명주기 관리"""
    # ... 다른 초기화 코드 ...

    # Startup: OAuth2PasswordBearer 초기화 (Swagger UI용)
    try:
        oauth2_scheme = get_oauth2_scheme()
        app.state.oauth2_scheme = oauth2_scheme
        logger.info("OAuth2Scheme initialized successfully")
    except RuntimeError as e:
        # KEYCLOAK_TOKEN_URL이 없으면 경고만 출력
        logger.warning(f"OAuth2Scheme initialization skipped: {str(e)}")

    yield

    # ... Shutdown 코드 ...

app = FastAPI(lifespan=lifespan)
```

### @authenticate_required 데코레이터의 자동 Security 추가

`@authenticate_required` 데코레이터는 함수 시그니처를 자동으로 수정하여 Swagger UI에 security 정보를 추가합니다.

#### 핵심 동작 원리

**⚠️ 중요**: 아래 코드 블록이 없으면 Swagger UI에 Authorize 버튼이 표시되지 않습니다!

FastAPI는 함수 시그니처에 `Depends(OAuth2PasswordBearer)`가 포함되어 있을 때만 OpenAPI 스키마에 security 정보를 자동으로 추가합니다. 따라서 데코레이터에서 함수 시그니처를 동적으로 수정하여 `token` 파라미터를 추가해야 합니다.

```python
# shared/decorators.py

import inspect
from fastapi import Depends
from dependencies import get_oauth2_scheme

def authenticate_required(func: Callable) -> Callable:
    """인증된 사용자만 접근 가능한 엔드포인트를 표시하는 데코레이터.

    Swagger UI에 Authorize 버튼을 자동으로 추가하기 위해
    함수에 OAuth2PasswordBearer dependency를 자동으로 추가합니다.

    Args:
        func: 데코레이터를 적용할 함수

    Returns:
        인증 검증 로직이 추가된 래퍼 함수
    """
    # Circular import 방지를 위한 지연 import
    from dependencies import get_oauth2_scheme

    # Swagger UI에 Authorize 버튼을 표시하기 위해 OAuth2PasswordBearer dependency 추가
    oauth2_scheme = get_oauth2_scheme()

    # 함수의 시그니처를 수정하여 token 파라미터 추가
    sig = inspect.signature(func)
    params = list(sig.parameters.values())

    # token 파라미터가 이미 있는지 확인
    has_token_param = any(p.name == "token" for p in params)

    # ⚠️ 이 코드 블록이 핵심입니다!
    # 이 코드가 없으면 FastAPI가 OpenAPI 스키마에 security 정보를 추가하지 않습니다.
    if not has_token_param:
        # token 파라미터를 추가 (Swagger UI용, 실제로는 사용하지 않음)
        token_param = inspect.Parameter(
            "token",
            inspect.Parameter.KEYWORD_ONLY,
            default=Depends(oauth2_scheme),  # ← 이 Depends가 Swagger UI 연결의 핵심!
            annotation=str | None,
        )
        params.append(token_param)
        new_sig = sig.replace(parameters=params)
        func.__signature__ = new_sig  # ← 함수 시그니처를 동적으로 수정

    @wraps(func)
    async def wrapper(*args, **kwargs) -> Any:
        # token 파라미터가 kwargs에 있으면 제거 (실제로는 사용하지 않음)
        kwargs.pop("token", None)

        authenticated_user = get_authenticated_user()

        if not authenticated_user:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="인증이 필요합니다",
                headers={"WWW-Authenticate": "Bearer"},
            )

        return await func(*args, **kwargs)

    return wrapper
```

#### 왜 이 코드가 필요한가?

1. **FastAPI의 OpenAPI 스키마 생성 방식**: FastAPI는 함수 시그니처를 분석하여 OpenAPI 스키마를 생성합니다. `Depends(OAuth2PasswordBearer)`가 함수 파라미터에 포함되어 있어야만 `components.securitySchemes`에 OAuth2 정보가 추가됩니다.

2. **데코레이터의 한계**: 일반적인 데코레이터는 함수 실행 시점에만 동작하지만, OpenAPI 스키마는 애플리케이션 시작 시점에 생성됩니다. 따라서 함수 시그니처를 수정해야 합니다.

3. **동적 시그니처 수정**: `inspect.Parameter`와 `func.__signature__`를 사용하여 함수 시그니처를 동적으로 수정합니다. 이렇게 하면 FastAPI가 함수를 분석할 때 `Depends(OAuth2PasswordBearer)`를 발견하고 security 정보를 추가합니다.

#### 검증 방법

이 코드가 제대로 작동하는지 확인하려면:

```bash
# OpenAPI 스키마에서 security 정보 확인
curl http://localhost:8000/openapi.json | jq '.paths["/user"].put.security'

# 출력 예시 (정상 작동 시)
[
  {
    "OAuth2PasswordBearer": []
  }
]

# 출력이 빈 배열 []이거나 null이면 시그니처 수정이 실패한 것입니다
```

### 사용 방법

#### 1. 엔드포인트에 데코레이터 적용

```python
# subdomains/user/interface/routers/user_router.py

from fastapi import APIRouter
from shared.decorators import authenticate_required

router = APIRouter()

@router.put("/user", response_model=UpdateUserResponse)
@authenticate_required
async def update_user(
    request: UpdateUserRequest,
    app_service: UserAppService = Depends(get_user_app_service),
) -> UpdateUserResponse:
    """사용자 정보 수정/생성 (Upsert)"""
    # token 파라미터를 명시적으로 추가할 필요 없음
    # @authenticate_required 데코레이터가 자동으로 Swagger UI에 security 추가
    ...
```

#### 2. Swagger UI에서 사용

1. `http://localhost:8000/docs` 접속
2. 우측 상단의 **"Authorize"** 버튼 클릭
3. 토큰 입력 필드에 Bearer 토큰 입력 (예: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`)
4. **"Authorize"** 버튼 클릭하여 인증 완료
5. 보호된 엔드포인트를 테스트할 수 있음

### OpenAPI 스키마 확인

Swagger UI에 security가 제대로 추가되었는지 확인하려면:

```bash
# OpenAPI 스키마에서 securitySchemes 확인
curl http://localhost:8000/openapi.json | jq '.components.securitySchemes'

# 특정 엔드포인트의 security 확인 (시그니처 수정이 제대로 되었는지 검증)
curl http://localhost:8000/openapi.json | jq '.paths["/user"].put.security'

# 출력 예시 (정상 작동 시)
# [
#   {
#     "OAuth2PasswordBearer": []
#   }
# ]
# 
# 출력이 빈 배열 []이거나 null이면 시그니처 수정이 실패한 것입니다
```

# 출력 예시
{
  "OAuth2PasswordBearer": {
    "type": "oauth2",
    "flows": {
      "password": {
        "scopes": {},
        "tokenUrl": "https://keycloak.example.com/realms/your-realm/protocol/openid-connect/token"
      }
    }
  }
}
```

### 주의사항

1. **환경 변수 필수**: `KEYCLOAK_TOKEN_URL`이 설정되지 않으면 OAuth2PasswordBearer 초기화가 실패합니다. 이 경우 Swagger UI에 Authorize 버튼이 표시되지 않습니다.

2. **토큰 파라미터 사용 금지**: `@authenticate_required` 데코레이터가 자동으로 추가하는 `token` 파라미터는 실제 함수에서 사용하지 않습니다. 데코레이터 내부에서 자동으로 제거됩니다.

3. **Middleware와의 관계**: Swagger UI의 Authorize 기능은 OpenAPI 스키마 생성용이며, 실제 인증은 `authentication_middleware`에서 처리됩니다.

---

## 사용 예제

### 예제 1: 기본 인증 확인

```python
# HTTP 요청
GET /api/me HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# 응답
{
    "user_id": "user123",
    "role": "user"
}
```

### 예제 2: 토큰 없이 요청

```python
# HTTP 요청 (Authorization 헤더 없음)
GET /api/protected HTTP/1.1

# 응답 (401 Unauthorized)
{
    "detail": "인증이 필요합니다"
}
```

### 예제 3: 권한 부족

```python
# HTTP 요청
DELETE /api/users/123 HTTP/1.1
Authorization: Bearer <일반_사용자_토큰>

# 응답 (403 Forbidden)
{
    "detail": "이 작업을 수행할 권한이 없습니다"
}
```

- [FastAPI 보안 공식 문서](https://fastapi.tiangolo.com/tutorial/security/)
- [JWT 표준 (RFC 7519)](https://tools.ietf.org/html/rfc7519)
- [Python ContextVar 공식 문서](https://docs.python.org/3/library/contextvars.html)
