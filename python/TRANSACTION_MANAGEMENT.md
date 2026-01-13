# Python 트랜잭션 관리 표준

문서 탐색: [네이밍 요약](naming_convention/naming_summary.md) · [테스트 전략](TESTING_STRATEGY.md)

Python에서는 Java의 `javax.sql` 같은 표준 트랜잭션 인터페이스가 없습니다. 따라서 우리는 **Transaction 패턴**으로 트랜잭션을 추상화하고, **DI(의존성 주입)**로 구현체를 주입합니다.

## 아키텍처 개요

```
┌─────────────────────────────────────────────────────────┐
│ Application Layer (비즈니스 오케스트레이션)              │
│ - UserAppService, ProductAppService 등                  │
│ - TransactionProtocol (추상화) + RepositoryProtocol 사용│
└────────────────────┬────────────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────────┐ ┌───▼──────────┐ ┌─▼──────────────┐
│Domain Layer │ │Shared/Core  │ │Interface       │
│(비즈니스)    │ │(기술 인프라)  │ │(HTTP/Router)   │
├─────────────┤ ├─────────────┤ │                │
│models/      │ │protocols/   │ │schemas/        │
│  User       │ │  Transaction│ │routers/        │
│protocols/   │ └─────────────┘ │                │
│  Repository │       ▲         └────────────────┘
│             │       │ impl
└─────────────┘ ┌─────┴────────────────────────────┐
                │ Infra Layer (SQLAlchemy)         │
                ├──────────────────────────────────┤
                │ infra/                           │
                │  transaction.py                  │
                │  repositories/                   │
                │    user_repository.py            │
                │    product_repository.py         │
                │                                  │
                │ shared/infra/                    │
                │  transaction.py (재사용)         │
                └──────────────────────────────────┘
```

## 폴더 구조

```
shared/
  protocols/
    transaction_protocol.py        # Transaction 인터페이스
  infra/
    transaction.py                 # SqlAlchemyTransaction 구현 (재사용)

domain/
  protocols/
    user_repository_protocol.py
    product_repository_protocol.py

infra/
  repositories/
    user_repository.py             # Repository 구현
    product_repository.py

application/
  services/
    user_app_service.py            # Transaction 주입받음
    product_app_service.py
```

## Readonly/Writable 트랜잭션 분리 (권장)

목표: 읽기 트래픽은 읽기 전용 트랜잭션으로, 쓰기 트래픽은 쓰기 트랜잭션으로 명확히 분리한다. 이는 추후 읽기 전용 리플리카(또는 read-only 세션 옵션)로 라우팅하기 위한 준비 작업이다.

- **타입 정의**: `TransactionMode = Literal["readonly", "writable"]` 를 `shared/protocols/transaction.py` 에서 정의하고, `TransactionProtocol` 에 `mode` 속성(default: "writable")을 둔다.
- **구현체 주입**: 트랜잭션 구현(`SQLAlchemyTransaction`, `PsycopgTransaction`)은 생성자에서 `mode`를 받는다.
- **DI 패턴**: 요청 단위로 동일 세션/커넥션을 공유하며 두 개의 팩토리를 주입한다.

```python
# core/dependencies.py (발췌)
session = await db_pool.get_session()

async def readonly_tx_factory() -> TransactionProtocol:
        return SQLAlchemyTransaction(session, mode="readonly")

async def writable_tx_factory() -> TransactionProtocol:
        return SQLAlchemyTransaction(session, mode="writable")

service = UserAppService(
        user_repository=SQLAlchemyUserRepository(session),
        readonly_tx_factory=readonly_tx_factory,
        writable_tx_factory=writable_tx_factory,
)
```

- **사용 규칙 (Application Service)**
    - **쿼리/조회**: `_run_readonly(...)` 로 감싸 `readonly_tx_factory` 사용 → 추후 read-replica로 라우팅 가능.
    - **커맨드/쓰기**: `_run_writable(...)` 로 감싸 `writable_tx_factory` 사용 → 기본 master/write DB로 라우팅.
    - 동일 요청 내에서 두 팩토리는 같은 세션/커넥션을 재사용하여 커넥션 폭주를 방지한다.

- **추후 확장**
    - 읽기 전용 트랜잭션에서만 `SET TRANSACTION READ ONLY` 또는 read-replica DNS로 전환.
    - Infra 레벨에서 `mode` 값을 확인해 라우팅/옵션 적용.

## Transaction Manager 패턴 (권장)

목표: Application Layer는 트랜잭션 생성 책임을 가지지 않고, `TransactionManager`에 위임한다. 트랜잭션은 `connection`(실제로는 SQLAlchemy `AsyncSession` 또는 psycopg `AsyncConnection`)을 노출해 Repository가 의존한다.

- **인터페이스** (`shared/protocols/transaction.py`)

```python
TransactionMode = Literal["readonly", "writable"]

class Connection(Protocol):
    pass  # AsyncSession | AsyncConnection

class TransactionProtocol(ABC):
    mode: TransactionMode
    @property
    @abstractmethod
    def connection(self) -> Connection: ...
    async def commit(self) -> None: ...
    async def rollback(self) -> None: ...

class TransactionManager(ABC):
    async def create_readonly_transaction(self) -> TransactionProtocol: ...
    async def create_writable_transaction(self) -> TransactionProtocol: ...
```

- **Repository 시그니처**: 모든 메서드는 `connection`을 첫 번째 인자로 받는다.

```python
class UserRepositoryProtocol(ABC):
    @abstractmethod
    async def find_by_id(self, conn: Connection, user_id: int) -> User | None: ...
    @abstractmethod
    async def add(self, conn: Connection, user: User) -> User: ...
```

- **Application Service 사용 패턴**: 트랜잭션 생성 → 컨텍스트에서 실행 → `tx.connection`을 Repository에 전달.

```python
class UserAppService:
    def __init__(self, repository: UserRepositoryProtocol, tx_manager: TransactionManager):
        self._repo = repository
        self._txm = tx_manager

    async def _run_readonly(self, fn):
        tx = await self._txm.create_readonly_transaction()
        async with tx:
            return await fn(tx.connection)

    async def get_user(self, user_id: int) -> UserDTO:
        async def _logic(conn):
            user = await self._repo.find_by_id(conn, user_id)
            if not user:
                raise NotFoundError("user")
            return UserDTO.from_domain(user)
        return await self._run_readonly(_logic)
```

- **DI 구성 예시** (`core/dependencies.py` 발췌)

```python
db_pool = DatabasePool()

def get_user_service() -> UserAppService:
    repository_type = os.getenv("REPOSITORY_TYPE", "sqlalchemy")
    if repository_type == "psycopg":
        tx_manager = PsycopgTransactionManager(db_pool)
        repo = PsycopgUserRepository()
    else:
        tx_manager = SQLAlchemyTransactionManager(db_pool)
        repo = SQLAlchemyUserRepository()
    return UserAppService(repo, tx_manager)
```

### 이점

1) **의존성 역전**: Application Layer는 구체 DB 세션 타입을 알 필요가 없음 (`Connection` 프로토콜로 추상화).
2) **리소스 일관성**: 한 트랜잭션 안에서 동일 커넥션/세션을 재사용하므로 커넥션 폭주 방지.
3) **교체 용이성**: SQLAlchemy ↔︎ psycopg 구현을 `TransactionManager` 스위치만으로 교체 가능.
4) **테스트 용이성**: 테스트에서는 `MockTransactionManager`로 트랜잭션 생성과 커밋/롤백 호출을 검증.

## 구현

### 1) Transaction 인터페이스 (shared/protocols/)

```python
# shared/protocols/transaction_protocol.py
from abc import ABC, abstractmethod
from typing import AsyncContextManager

class TransactionProtocol(ABC):
    """트랜잭션 경계 관리 포트
    
    Async context manager로 사용:
    async with transaction:
        await repository.save(obj)
        await transaction.commit()
    """
    
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

### 2) Transaction 구현 (shared/infra/)

```python
# shared/infra/transaction.py
from typing import Optional
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, AsyncEngine
from sqlalchemy.orm import sessionmaker
from shared.protocols.transaction_protocol import TransactionProtocol

class SqlAlchemyTransaction(TransactionProtocol):
    """SQLAlchemy 기반 Transaction 구현 (재사용 가능)"""
    
    def __init__(self, session: AsyncSession):
        self.session = session
    
    async def commit(self) -> None:
        await self.session.commit()
    
    async def rollback(self) -> None:
        await self.session.rollback()
```

### 3) Repository 포트 (domain/protocols/)

```python
# domain/protocols/user_repository_protocol.py
from abc import ABC, abstractmethod
from domain.models.user import User

class UserRepositoryProtocol(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User:
        """사용자 조회"""
        pass
    
    @abstractmethod
    async def save(self, user: User) -> User:
        """사용자 저장 (INSERT/UPDATE)"""
        pass
    
    @abstractmethod
    async def save_many(self, users: list[User]) -> None:
        """대량 저장"""
        pass
    
    @abstractmethod
    async def delete(self, user_id: int) -> None:
        """사용자 삭제"""
        pass
```

### 4) Repository 구현 (infra/repositories/)

```python
# infra/repositories/user_repository.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update
from domain.models.user import User
from domain.protocols.user_repository_protocol import UserRepositoryProtocol
from infra.orm.user_entity import UserEntity

class UserRepository(UserRepositoryProtocol):
    def __init__(self, session: AsyncSession):
        self.session = session
    
    async def get_by_id(self, user_id: int) -> User:
        result = await self.session.execute(
            select(UserEntity).where(UserEntity.user_id == user_id)
        )
        entity = result.scalar_one_or_none()
        if not entity:
            raise ValueError(f"User {user_id} not found")
        return self._entity_to_domain(entity)
    
    async def save(self, user: User) -> User:
        entity = self._domain_to_entity(user)
        self.session.add(entity)
        await self.session.flush()  # 즉시 flush (commit 전)
        return self._entity_to_domain(entity)
    
    async def save_many(self, users: list[User]) -> None:
        """벌크 업데이트 (효율적)"""
        entities = [self._domain_to_entity(u) for u in users]
        # 세션에 추가
        for entity in entities:
            self.session.add(entity)
        await self.session.flush()
    
    async def delete(self, user_id: int) -> None:
        await self.session.execute(
            select(UserEntity).where(UserEntity.user_id == user_id)
        )
        await self.session.delete(
            select(UserEntity).where(UserEntity.user_id == user_id)
        )
    
    def _domain_to_entity(self, user: User) -> UserEntity:
        return UserEntity(
            user_id=user.user_id,
            email=user.email,
            name=user.name,
        )
    
    def _entity_to_domain(self, entity: UserEntity) -> User:
        return User(user_id=entity.user_id, email=entity.email, name=entity.name)
```

### 5) Application Service (application/services/)

```python
# application/services/user_app_service.py
from application.dtos.user_dto import UpdateUserCommand, UpdateUserCommandResult
from application.dtos.user_bulk_dto import UpdateUserBulkCommand, UpdateUserBulkResult
from domain.protocols.user_repository_protocol import UserRepositoryProtocol
from shared.protocols.transaction_protocol import TransactionProtocol

class UserAppService:
    def __init__(self, 
                 user_repo: UserRepositoryProtocol,
                 transaction: TransactionProtocol):
        self.user_repo = user_repo
        self.transaction = transaction
    
    async def update_user(self, cmd: UpdateUserCommand) -> UpdateUserCommandResult:
        """단건 업데이트"""
        async with self.transaction:
            user = await self.user_repo.get_by_id(cmd.user_id)
            updated = user.change_profile(email=cmd.email, name=cmd.name)
            saved = await self.user_repo.save(updated)
        
        return UpdateUserCommandResult(
            user_id=saved.user_id,
            email=saved.email,
            name=saved.name,
        )
    
    async def update_users_bulk(self, cmd: UpdateUserBulkCommand) -> UpdateUserBulkResult:
        """대량 업데이트 (한 트랜잭션)"""
        success_ids = []
        failed_ids = []
        
        async with self.transaction:
            users_to_save = []
            
            for item in cmd.items:
                try:
                    user = await self.user_repo.get_by_id(item.user_id)
                    updated = user.change_profile(email=item.email, name=item.name)
                    users_to_save.append(updated)
                    success_ids.append(item.user_id)
                except Exception as e:
                    failed_ids.append(item.user_id)
            
            # 모든 변경을 한 번에 저장
            if users_to_save:
                await self.user_repo.save_many(users_to_save)
        
        return UpdateUserBulkResult(success_ids=success_ids, failed_ids=failed_ids)
```

## DI 설정 (main.py / bootstrap)

```python
# main.py
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from shared.infra.transaction import SqlAlchemyTransaction
from infra.repositories.user_repository import UserRepository
from infra.repositories.product_repository import ProductRepository
from application.services.user_app_service import UserAppService
from application.services.product_app_service import ProductAppService

# SQLAlchemy 엔진/세션 팩토리 설정
DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"
engine = create_async_engine(DATABASE_URL)
async_session_maker = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_session() -> AsyncSession:
    async with async_session_maker() as session:
        yield session

async def get_transaction(session: AsyncSession = Depends(get_session)) -> TransactionProtocol:
    return SqlAlchemyTransaction(session)

# 서비스 초기화
async def get_user_service(
    transaction: TransactionProtocol = Depends(get_transaction),
    session: AsyncSession = Depends(get_session),
) -> UserAppService:
    repo = UserRepository(session)
    return UserAppService(repo, transaction)

app = FastAPI()

# 라우터에서 주입받아 사용
@app.put("/users/{user_id}")
async def update_user(
    user_id: int,
    cmd: UpdateUserCommand,
    service: UserAppService = Depends(get_user_service),
) -> UpdateUserCommandResult:
    return await service.update_user(UpdateUserCommand(user_id=user_id, **cmd.dict()))
```

## 사용 패턴

### 단건 작업
```python
async with transaction:
    user = await repo.get_by_id(1)
    updated = user.change_profile(email="new@example.com", name="New Name")
    saved = await repo.save(updated)
# 자동 commit (예외 없음)
```

### 대량 작업 (한 트랜잭션)
```python
async with transaction:
    users = [await repo.get_by_id(i) for i in [1, 2, 3]]
    updated = [u.change_profile(...) for u in users]
    await repo.save_many(updated)
# 자동 commit
```

### 예외 발생 시 자동 rollback
```python
async with transaction:
    user = await repo.get_by_id(1)
    # ... 비즈니스 로직
    if error_condition:
        raise BusinessError()  # 자동 rollback
```

## 모범 사례

1. **한 트랜잭션 = 한 Transaction 컨텍스트**
   ```python
   async with transaction:
       # 이 블록 내의 모든 작업이 하나의 트랜잭션
   ```

2. **Repository는 Repository 포트만 따름**
   - Transaction 메서드를 직접 호출하지 않음
   - 트랜잭션 관리는 Application Layer의 책임

3. **예외 처리는 Application Service에서**
   ```python
   async with transaction:
       try:
           await repo.save(obj)
       except IntegrityError:
           # Application 로직으로 처리
   ```

4. **flush() vs commit() 이해**
   - `flush()`: DB로 전송하되 아직 커밋 안 됨 (ID 생성 등 필요)
   - `commit()`: 트랜잭션 확정

5. **여러 리포지토리를 한 트랜잭션에서**
   ```python
   async with transaction:
       user = await user_repo.save(new_user)
       order = await order_repo.save(new_order)  # 같은 세션
       # 둘 다 성공하거나 둘 다 롤백
   ```

## 버전 이력

- 1.0.1 (2026-01-11): UnitOfWork 용어를 Transaction으로 변경
- 1.0.0 (2026-01-11): 초기 Transaction 패턴 표준 정의
