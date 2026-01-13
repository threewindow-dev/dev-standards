# Python Database Layer 표준

문서 탐색: [네이밍 요약](naming_convention/naming_summary.md) · [테스트 전략](TESTING_STRATEGY.md) · [FastAPI 개발](FASTAPI_DEVELOPMENT_STANDARDS.md)

이 문서는 Python 백엔드에서 데이터베이스 계층(Database Layer)의 표준을 정의합니다. Connection Pool 관리, Transaction 패턴, Repository 구현을 포함합니다.

## 목차

1. [아키텍처 개요](#아키텍처-개요)
2. [핵심 추상화 계층](#핵심-추상화-계층)
   - [Connection (Protocol)](#connection-protocol)
   - [DatabasePool (ABC)](#databasepool-abc)
   - [TransactionManager (ABC)](#transactionmanager-abc)
   - [TransactionProtocol (ABC)](#transactionprotocol-abc)
3. [Transaction Management](#transaction-management)
   - [Readonly/Writable 분리](#readonlywritable-트랜잭션-분리)
   - [Transaction Manager 패턴](#transaction-manager-패턴)
4. [Repository Pattern](#repository-pattern)
5. [구현 가이드](#구현-가이드)
6. [DI 설정](#di-설정)
7. [모범 사례](#모범-사례)

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────┐
│ Application Layer (비즈니스 오케스트레이션)              │
│ - UserAppService, ProductAppService 등                  │
│ - TransactionManager + RepositoryProtocol 사용          │
└────────────────────┬────────────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────────┐ ┌───▼──────────┐ ┌─▼──────────────┐
│Domain Layer │ │Shared/Core  │ │Interface       │
│(비즈니스)    │ │(기술 인프라)  │ │(HTTP/Router)   │
├─────────────┤ ├─────────────┤ │                │
│models/      │ │protocols/   │ │schemas/        │
│  User       │ │  Connection │ │routers/        │
│protocols/   │ │  DatabasePool│ │                │
│  Repository │ │  Transaction│ └────────────────┘
└─────────────┘ └──────┬──────┘
                       │ impl
                ┌──────▼──────────────────────────┐
                │ Infra Layer                     │
                ├─────────────────────────────────┤
                │ shared/infra/database.py        │
                │  - DatabasePool 구현체          │
                │  - TransactionManager 구현체    │
                │                                 │
                │ subdomains/user/infra/          │
                │  repositories/                  │
                │    user_repository.py           │
                └─────────────────────────────────┘
```

### 폴더 구조

```
shared/
  protocols/
    database.py                    # DatabasePool 인터페이스
    transaction.py                 # Transaction 인터페이스
  infra/
    database.py                    # DatabasePool + TransactionManager 구현

subdomains/
  user/
    domain/
      protocols/
        user_repository.py         # Repository 인터페이스
      models/
        user.py                    # Domain Entity
    infra/
      repositories/
        user_repository.py         # Repository 구현
    application/
      services/
        user_app_service.py        # Application Service
```

## 핵심 추상화 계층

### Connection (Protocol)

`Connection`은 데이터베이스 연결의 범용 타입 힌트입니다. SQLAlchemy의 `AsyncSession`이나 psycopg의 `AsyncConnection` 등 실제 구현체를 추상화합니다.

```python
# shared/protocols/transaction.py
from typing import Protocol

class Connection(Protocol):
    """데이터베이스 연결 추상화
    
    실제 타입: AsyncSession | AsyncConnection
    - SQLAlchemy: AsyncSession (ORM 기반)
    - psycopg: AsyncConnection (raw SQL 기반)
    """
    pass
```

**사용 목적:**
- Repository 메서드 시그니처에서 구체적인 DB 라이브러리에 의존하지 않도록 함
- Application/Domain Layer가 인프라 구현 세부사항을 모르도록 격리
- 타입 체커에서 `AsyncSession | AsyncConnection` 대신 간결한 타입 사용

**Repository에서 사용:**
```python
class UserRepositoryProtocol(ABC):
    @abstractmethod
    async def find_by_id(self, conn: Connection, user_id: int) -> User | None: ...
    
    @abstractmethod
    async def add(self, conn: Connection, user: User) -> User: ...
```

### DatabasePool (ABC)

`DatabasePool`은 데이터베이스 연결 풀 관리를 추상화합니다. 연결 생성, 초기화, 종료를 담당하며, readonly/writable 모드를 지원합니다.

```python
# shared/protocols/database.py
from abc import ABC, abstractmethod
from contextlib import asynccontextmanager
from typing import AsyncGenerator

class DatabasePool(ABC):
    """데이터베이스 연결 풀 추상 인터페이스
    
    구현체:
    - SQLAlchemyDatabasePool: AsyncEngine + AsyncSession 기반
    - PsycopgDatabasePool: AsyncConnection 직접 관리
    """
    
    @abstractmethod
    async def initialize(self) -> None:
        """연결 풀 초기화 (엔진/팩토리 생성)"""
        pass
    
    @abstractmethod
    async def close(self) -> None:
        """연결 풀 종료 (리소스 정리)"""
        pass
    
    @abstractmethod
    async def get_connection(self, mode: TransactionMode = "writable") -> Connection:
        """연결 획득 (세션 또는 커넥션)
        
        Args:
            mode: "readonly" (읽기 전용) | "writable" (쓰기 가능)
        
        Returns:
            AsyncSession | AsyncConnection
        """
        pass
    
    @abstractmethod
    @asynccontextmanager
    async def connection(self, mode: TransactionMode = "writable") -> AsyncGenerator[Connection, None]:
        """연결 컨텍스트 매니저
        
        자동으로 연결 획득 및 종료 처리:
        async with pool.connection() as conn:
            # 작업 수행
        # 자동 close()
        """
        pass
```

**팩토리 패턴:**

```python
# shared/infra/database.py
def db_pool_factory(repository_type: str = "sqlalchemy") -> DatabasePool:
    """환경 변수 기반 DatabasePool 구현체 생성
    
    Args:
        repository_type: "sqlalchemy" | "psycopg"
    
    Returns:
        SQLAlchemyDatabasePool | PsycopgDatabasePool
    
    Example:
        # main.py
        db_pool = db_pool_factory(os.getenv("REPOSITORY_TYPE", "sqlalchemy"))
        await db_pool.initialize()
    """
    if repository_type == "sqlalchemy":
        return SQLAlchemyDatabasePool()
    elif repository_type == "psycopg":
        return PsycopgDatabasePool()
    else:
        raise ValueError(f"Unknown repository type: {repository_type}")
```

**사용 예시:**

```python
# main.py (애플리케이션 시작)
import os
from shared.infra.database import db_pool_factory

db_pool = db_pool_factory(os.getenv("REPOSITORY_TYPE", "sqlalchemy"))
await db_pool.initialize()

# 연결 사용
async with db_pool.connection(mode="readonly") as conn:
    # readonly 작업
    result = await conn.execute(select(...))

# 애플리케이션 종료 시
await db_pool.close()
```

**환경 변수:**
```bash
# SQLAlchemy 사용 (기본값)
REPOSITORY_TYPE=sqlalchemy

# psycopg 사용
REPOSITORY_TYPE=psycopg

# Readonly replica 활성화 (선택)
DB_READONLY_ENABLED=true
DB_READONLY_HOST=read-replica.example.com
DB_READONLY_PORT=5432
```

### TransactionManager (ABC)

`TransactionManager`는 트랜잭션 생성을 추상화합니다. Application Service는 이를 통해 readonly/writable 트랜잭션을 생성하며, 구체적인 DB 라이브러리를 알 필요가 없습니다.

```python
# shared/protocols/transaction.py
from abc import ABC, abstractmethod

class TransactionManager(ABC):
    """트랜잭션 생성 팩토리
    
    구현체:
    - SQLAlchemyTransactionManager
    - PsycopgTransactionManager
    """
    
    @abstractmethod
    async def create_readonly_transaction(self) -> TransactionProtocol:
        """읽기 전용 트랜잭션 생성
        
        Returns:
            mode="readonly" TransactionProtocol
        """
        pass
    
    @abstractmethod
    async def create_writable_transaction(self) -> TransactionProtocol:
        """쓰기 트랜잭션 생성
        
        Returns:
            mode="writable" TransactionProtocol
        """
        pass
```

**구현 예시:**

```python
# shared/infra/database.py
class SQLAlchemyTransactionManager(TransactionManager):
    """SQLAlchemy 기반 트랜잭션 매니저"""
    
    def __init__(self, db_pool: DatabasePool):
        self._db_pool = db_pool
    
    async def create_readonly_transaction(self) -> TransactionProtocol:
        session = await self._db_pool.get_connection(mode="readonly")
        return SQLAlchemyTransaction(session, mode="readonly")
    
    async def create_writable_transaction(self) -> TransactionProtocol:
        session = await self._db_pool.get_connection(mode="writable")
        return SQLAlchemyTransaction(session, mode="writable")
```

**Application Service에서 사용:**

```python
class UserAppService:
    def __init__(self, repository: UserRepositoryProtocol, tx_manager: TransactionManager):
        self._repo = repository
        self._txm = tx_manager

    async def _run_readonly(self, fn):
        """읽기 전용 트랜잭션 실행 헬퍼"""
        tx = await self._txm.create_readonly_transaction()
        async with tx:
            return await fn(tx.connection)
    
    async def _run_writable(self, fn):
        """쓰기 트랜잭션 실행 헬퍼"""
        tx = await self._txm.create_writable_transaction()
        async with tx:
            return await fn(tx.connection)

    async def get_user(self, user_id: int) -> UserDTO:
        """조회 작업 - readonly 트랜잭션"""
        async def _logic(conn):
            user = await self._repo.find_by_id(conn, user_id)
            if not user:
                raise UserNotFoundError(user_id)
            return UserDTO.from_domain(user)
        return await self._run_readonly(_logic)
    
    async def create_user(self, cmd: CreateUserCommand) -> UserDTO:
        """생성 작업 - writable 트랜잭션"""
        async def _logic(conn):
            user = User.create(
                username=cmd.username,
                email=cmd.email,
                full_name=cmd.full_name
            )
            saved = await self._repo.add(conn, user)
            return UserDTO.from_domain(saved)
        return await self._run_writable(_logic)
```

### TransactionProtocol (ABC)

`TransactionProtocol`은 트랜잭션의 생명주기(commit/rollback)를 관리합니다.

```python
# shared/protocols/transaction.py
from abc import ABC, abstractmethod
from typing import Literal

TransactionMode = Literal["readonly", "writable"]

class TransactionProtocol(ABC):
    """트랜잭션 경계 관리
    
    Context manager로 자동 commit/rollback:
    async with transaction:
        # 작업 수행
    # 정상: 자동 commit
    # 예외: 자동 rollback
    """
    
    mode: TransactionMode
    
    @property
    @abstractmethod
    def connection(self) -> Connection:
        """트랜잭션이 관리하는 DB 연결"""
        pass
    
    async def __aenter__(self) -> "TransactionProtocol":
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if exc_type is not None:
            await self.rollback()
        else:
            await self.commit()
    
    @abstractmethod
    async def commit(self) -> None:
        """트랜잭션 커밋"""
        pass
    
    @abstractmethod
    async def rollback(self) -> None:
        """트랜잭션 롤백"""
        pass
```

**구현 예시:**

```python
# shared/infra/database.py
class SQLAlchemyTransaction(TransactionProtocol):
    """SQLAlchemy 기반 트랜잭션"""
    
    def __init__(self, session: AsyncSession, mode: TransactionMode = "writable"):
        self._session = session
        self.mode = mode
    
    @property
    def connection(self) -> Connection:
        return self._session
    
    async def commit(self) -> None:
        await self._session.commit()
    
    async def rollback(self) -> None:
        await self._session.rollback()
```

## Transaction Management

### Readonly/Writable 트랜잭션 분리

**목표:** 읽기 트래픽은 읽기 전용 트랜잭션으로, 쓰기 트래픽은 쓰기 트랜잭션으로 명확히 분리합니다. 이는 추후 읽기 전용 리플리카(read replica)로 라우팅하기 위한 준비 작업입니다.

**핵심 원칙:**
1. **쿼리/조회 작업**: `_run_readonly()` 헬퍼 사용 → `create_readonly_transaction()`
2. **커맨드/변경 작업**: `_run_writable()` 헬퍼 사용 → `create_writable_transaction()`
3. 동일 요청 내에서도 명시적으로 mode를 분리하여 의도를 명확히 함

**장점:**
- 추후 read-replica 분리 시 코드 변경 최소화
- 의도가 명확하여 버그 방지 (실수로 readonly에서 write 시도 방지 가능)
- 성능 최적화 기회 (DB 엔진에서 readonly 힌트 활용)

### Transaction Manager 패턴

**이점:**

1. **의존성 역전**: Application Layer는 구체 DB 세션 타입을 알 필요 없음 (`Connection` 프로토콜로 추상화)
2. **리소스 일관성**: 한 트랜잭션 안에서 동일 커넥션/세션을 재사용하므로 커넥션 폭주 방지
3. **교체 용이성**: SQLAlchemy ↔︎ psycopg 구현을 `TransactionManager` 스위치만으로 교체 가능
4. **테스트 용이성**: 테스트에서는 `MockTransactionManager`로 트랜잭션 생성과 커밋/롤백 검증

## Repository Pattern

### Repository 인터페이스

모든 Repository 메서드는 `Connection`을 첫 번째 인자로 받습니다.

```python
# subdomains/user/domain/protocols/user_repository.py
from abc import ABC, abstractmethod
from shared.protocols.transaction import Connection
from subdomains.user.domain.models.user import User

class UserRepositoryProtocol(ABC):
    @abstractmethod
    async def find_by_id(self, conn: Connection, user_id: int) -> User | None:
        """사용자 ID로 조회"""
        pass
    
    @abstractmethod
    async def find_by_username(self, conn: Connection, username: str) -> User | None:
        """사용자명으로 조회"""
        pass
    
    @abstractmethod
    async def find_all(self, conn: Connection, skip: int = 0, limit: int = 100) -> list[User]:
        """사용자 목록 조회 (페이징)"""
        pass
    
    @abstractmethod
    async def add(self, conn: Connection, user: User) -> User:
        """사용자 추가"""
        pass
    
    @abstractmethod
    async def update(self, conn: Connection, user: User) -> User:
        """사용자 수정"""
        pass
    
    @abstractmethod
    async def remove(self, conn: Connection, user_id: int) -> None:
        """사용자 삭제"""
        pass
    
    @abstractmethod
    async def exists_by_username(self, conn: Connection, username: str) -> bool:
        """사용자명 존재 여부"""
        pass
    
    @abstractmethod
    async def exists_by_email(self, conn: Connection, email: str) -> bool:
        """이메일 존재 여부"""
        pass
```

### Repository 구현

```python
# subdomains/user/infra/repositories/user_repository.py
from sqlalchemy import select, delete
from sqlalchemy.ext.asyncio import AsyncSession
from subdomains.user.domain.models.user import User
from subdomains.user.domain.protocols.user_repository import UserRepositoryProtocol
from subdomains.user.infra.models.user_orm import UserORM
from shared.errors import InfraError

class SQLAlchemyUserRepository(UserRepositoryProtocol):
    """SQLAlchemy 기반 사용자 저장소"""
    
    async def find_by_id(self, conn: AsyncSession, user_id: int) -> User | None:
        result = await conn.execute(
            select(UserORM).where(UserORM.id == user_id)
        )
        orm = result.scalar_one_or_none()
        return self._to_domain(orm) if orm else None
    
    async def add(self, conn: AsyncSession, user: User) -> User:
        orm = self._to_orm(user)
        conn.add(orm)
        await conn.flush()  # ID 생성
        user.id = orm.id
        return user
    
    async def update(self, conn: AsyncSession, user: User) -> User:
        orm = await conn.get(UserORM, user.id)
        if not orm:
            raise InfraError(f"USER_NOT_FOUND: {user.id}")
        
        orm.username = user.username
        orm.email = user.email
        orm.full_name = user.full_name
        await conn.flush()
        return user
    
    async def remove(self, conn: AsyncSession, user_id: int) -> None:
        await conn.execute(delete(UserORM).where(UserORM.id == user_id))
        await conn.flush()
    
    def _to_domain(self, orm: UserORM) -> User:
        return User(
            id=orm.id,
            username=orm.username,
            email=orm.email,
            full_name=orm.full_name,
            created_at=orm.created_at
        )
    
    def _to_orm(self, user: User) -> UserORM:
        return UserORM(
            id=user.id,
            username=user.username,
            email=user.email,
            full_name=user.full_name,
            created_at=user.created_at
        )
```

## 구현 가이드

### 1. DatabasePool 구현

```python
# shared/infra/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from shared.protocols.database import DatabasePool

class SQLAlchemyDatabasePool(DatabasePool):
    """SQLAlchemy 기반 데이터베이스 연결 풀"""
    
    def __init__(self):
        self._engine_write = None
        self._engine_readonly = None
        self._session_factory_write = None
        self._session_factory_readonly = None
    
    async def initialize(self) -> None:
        """엔진 및 세션 팩토리 초기화"""
        dsn = os.getenv("DATABASE_URL")
        async_dsn = dsn.replace("postgresql://", "postgresql+asyncpg://")
        
        self._engine_write = create_async_engine(
            async_dsn,
            pool_size=int(os.getenv("DB_POOL_SIZE", "5")),
            max_overflow=int(os.getenv("DB_MAX_OVERFLOW", "10")),
            pool_pre_ping=True,
        )
        self._session_factory_write = async_sessionmaker(
            self._engine_write,
            class_=AsyncSession,
            expire_on_commit=False,
        )
        
        # Readonly 설정 (선택)
        if os.getenv("DB_READONLY_ENABLED") == "true":
            readonly_dsn = os.getenv("DB_READONLY_URL", async_dsn)
            self._engine_readonly = create_async_engine(readonly_dsn, ...)
            self._session_factory_readonly = async_sessionmaker(...)
        else:
            self._engine_readonly = self._engine_write
            self._session_factory_readonly = self._session_factory_write
    
    async def close(self) -> None:
        """엔진 종료"""
        if self._engine_write:
            await self._engine_write.dispose()
        if self._engine_readonly and self._engine_readonly != self._engine_write:
            await self._engine_readonly.dispose()
    
    async def get_connection(self, mode: TransactionMode = "writable") -> AsyncSession:
        factory = (self._session_factory_readonly if mode == "readonly" 
                   else self._session_factory_write)
        return factory()
    
    @asynccontextmanager
    async def connection(self, mode: TransactionMode = "writable"):
        session = await self.get_connection(mode=mode)
        try:
            yield session
        finally:
            await session.close()
```

### 2. Application Service 구현

```python
# subdomains/user/application/services/user_app_service.py
from subdomains.user.domain.protocols.user_repository import UserRepositoryProtocol
from subdomains.user.application.dtos.user_dto import UserDTO, CreateUserCommand
from shared.protocols.transaction import TransactionManager
from shared.errors import UserNotFoundError, ValidationError

class UserAppService:
    def __init__(
        self,
        user_repository: UserRepositoryProtocol,
        transaction_manager: TransactionManager,
    ):
        self._repo = user_repository
        self._txm = transaction_manager
    
    async def _run_readonly(self, fn):
        tx = await self._txm.create_readonly_transaction()
        async with tx:
            return await fn(tx.connection)
    
    async def _run_writable(self, fn):
        tx = await self._txm.create_writable_transaction()
        async with tx:
            return await fn(tx.connection)
    
    async def get_user(self, user_id: int) -> UserDTO:
        """사용자 조회 (readonly)"""
        async def _logic(conn):
            user = await self._repo.find_by_id(conn, user_id)
            if not user:
                raise UserNotFoundError(user_id)
            return UserDTO.from_domain(user)
        return await self._run_readonly(_logic)
    
    async def list_users(self, skip: int = 0, limit: int = 100) -> list[UserDTO]:
        """사용자 목록 조회 (readonly)"""
        async def _logic(conn):
            users = await self._repo.find_all(conn, skip=skip, limit=limit)
            return [UserDTO.from_domain(u) for u in users]
        return await self._run_readonly(_logic)
    
    async def create_user(self, cmd: CreateUserCommand) -> UserDTO:
        """사용자 생성 (writable)"""
        async def _logic(conn):
            # 중복 체크
            if await self._repo.exists_by_username(conn, cmd.username):
                raise ValidationError("USERNAME_ALREADY_EXISTS")
            if await self._repo.exists_by_email(conn, cmd.email):
                raise ValidationError("EMAIL_ALREADY_EXISTS")
            
            # 도메인 엔티티 생성
            user = User.create(
                username=cmd.username,
                email=cmd.email,
                full_name=cmd.full_name
            )
            
            # 저장
            saved = await self._repo.add(conn, user)
            return UserDTO.from_domain(saved)
        
        return await self._run_writable(_logic)
    
    async def update_user(self, user_id: int, full_name: str) -> UserDTO:
        """사용자 수정 (writable)"""
        async def _logic(conn):
            user = await self._repo.find_by_id(conn, user_id)
            if not user:
                raise UserNotFoundError(user_id)
            
            user.change_full_name(full_name)
            updated = await self._repo.update(conn, user)
            return UserDTO.from_domain(updated)
        
        return await self._run_writable(_logic)
    
    async def delete_user(self, user_id: int) -> None:
        """사용자 삭제 (writable)"""
        async def _logic(conn):
            user = await self._repo.find_by_id(conn, user_id)
            if not user:
                raise UserNotFoundError(user_id)
            
            await self._repo.remove(conn, user_id)
        
        await self._run_writable(_logic)
```

## DI 설정

### FastAPI Dependencies

```python
# core/dependencies.py
import os
from shared.protocols.database import DatabasePool
from shared.infra.database import (
    db_pool_factory,
    SQLAlchemyTransactionManager,
    PsycopgTransactionManager,
)
from subdomains.user.infra.repositories import (
    SQLAlchemyUserRepository,
    PsycopgUserRepository,
)
from subdomains.user.application.services.user_app_service import UserAppService

# DatabasePool 전역 인스턴스
_db_pool: DatabasePool | None = None

def set_db_pool(pool: DatabasePool) -> None:
    """main.py에서 초기화 시 호출"""
    global _db_pool
    _db_pool = pool

def get_db_pool() -> DatabasePool:
    if _db_pool is None:
        raise RuntimeError("DatabasePool not initialized")
    return _db_pool

# Application Service 의존성
async def get_user_app_service() -> UserAppService:
    """UserAppService DI"""
    db_pool = get_db_pool()
    repository_type = os.getenv("REPOSITORY_TYPE", "sqlalchemy")
    
    if repository_type == "sqlalchemy":
        tx_manager = SQLAlchemyTransactionManager(db_pool)
        repository = SQLAlchemyUserRepository()
        return UserAppService(repository, tx_manager)
    else:
        tx_manager = PsycopgTransactionManager(db_pool)
        repository = PsycopgUserRepository()
        return UserAppService(repository, tx_manager)
```

### Application Lifecycle

```python
# main.py
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI
from core.dependencies import set_db_pool
from shared.infra.database import db_pool_factory

@asynccontextmanager
async def lifespan(app: FastAPI):
    """애플리케이션 생명주기 관리"""
    # Startup
    db_pool = db_pool_factory(os.getenv("REPOSITORY_TYPE", "sqlalchemy"))
    await db_pool.initialize()
    set_db_pool(db_pool)
    
    yield
    
    # Shutdown
    await db_pool.close()

app = FastAPI(lifespan=lifespan)

# Router 등록
from subdomains.user.interface.routers import user_router
app.include_router(user_router.router, prefix="/api")
```

## 모범 사례

### 1. Transaction 범위 최소화

**권장:**
```python
async def create_user(self, cmd: CreateUserCommand) -> UserDTO:
    async def _logic(conn):
        # 모든 로직을 트랜잭션 내부에
        user = User.create(...)
        saved = await self._repo.add(conn, user)
        return UserDTO.from_domain(saved)
    return await self._run_writable(_logic)
```

**비권장:**
```python
async def create_user(self, cmd: CreateUserCommand) -> UserDTO:
    user = User.create(...)  # 트랜잭션 밖에서 생성
    
    async def _logic(conn):
        saved = await self._repo.add(conn, user)  # 만 트랜잭션 내부
        return UserDTO.from_domain(saved)
    return await self._run_writable(_logic)
```

### 2. Readonly vs Writable 명확히 분리

```python
# ✅ 조회는 readonly
async def get_user(self, user_id: int) -> UserDTO:
    return await self._run_readonly(...)

async def list_users(self, skip: int, limit: int) -> list[UserDTO]:
    return await self._run_readonly(...)

# ✅ 변경은 writable
async def create_user(self, cmd: CreateUserCommand) -> UserDTO:
    return await self._run_writable(...)

async def update_user(self, user_id: int, full_name: str) -> UserDTO:
    return await self._run_writable(...)
```

### 3. 예외 처리

```python
async def create_user(self, cmd: CreateUserCommand) -> UserDTO:
    async def _logic(conn):
        # 중복 체크에서 예외 발생 → 자동 rollback
        if await self._repo.exists_by_username(conn, cmd.username):
            raise ValidationError("USERNAME_ALREADY_EXISTS")
        
        user = User.create(...)
        saved = await self._repo.add(conn, user)
        return UserDTO.from_domain(saved)
    
    # 예외가 발생하면 _run_writable이 자동으로 rollback
    return await self._run_writable(_logic)
```

### 4. 복수 Repository 사용

```python
class OrderAppService:
    def __init__(
        self,
        order_repo: OrderRepositoryProtocol,
        user_repo: UserRepositoryProtocol,
        tx_manager: TransactionManager,
    ):
        self._order_repo = order_repo
        self._user_repo = user_repo
        self._txm = tx_manager
    
    async def create_order(self, cmd: CreateOrderCommand) -> OrderDTO:
        async def _logic(conn):
            # 같은 conn을 여러 repository에 전달
            user = await self._user_repo.find_by_id(conn, cmd.user_id)
            if not user:
                raise UserNotFoundError(cmd.user_id)
            
            order = Order.create(user_id=user.id, items=cmd.items)
            saved = await self._order_repo.add(conn, order)
            
            # 모두 성공하거나 모두 rollback
            return OrderDTO.from_domain(saved)
        
        return await self._run_writable(_logic)
```

### 5. flush() 활용

```python
async def add(self, conn: AsyncSession, user: User) -> User:
    orm = self._to_orm(user)
    conn.add(orm)
    await conn.flush()  # DB에 전송하여 ID 생성 (아직 commit 안 함)
    user.id = orm.id     # 생성된 ID를 domain 엔티티에 반영
    return user
```

### 6. 대량 작업 최적화

```python
async def bulk_update_users(self, commands: list[UpdateUserCommand]) -> BulkResult:
    async def _logic(conn):
        success_ids = []
        failed_ids = []
        
        for cmd in commands:
            try:
                user = await self._repo.find_by_id(conn, cmd.user_id)
                if user:
                    user.change_full_name(cmd.full_name)
                    await self._repo.update(conn, user)
                    success_ids.append(cmd.user_id)
            except Exception:
                failed_ids.append(cmd.user_id)
        
        # 한 번에 commit
        return BulkResult(success_ids=success_ids, failed_ids=failed_ids)
    
    return await self._run_writable(_logic)
```

## 버전 이력

- 2.0.0 (2026-01-13): DATABASE_STANDARDS.md로 재구성, 전체 DB Layer 표준 통합
- 1.1.0 (2026-01-13): DatabasePool, Connection, TransactionManager 추상화 계층 표준 추가
- 1.0.1 (2026-01-11): UnitOfWork 용어를 Transaction으로 변경
- 1.0.0 (2026-01-11): 초기 Transaction 패턴 표준 정의
