# Python Testing Strategy

문서 탐색: [네이밍 요약](naming_convention/naming_summary.md) · [트랜잭션 관리](TRANSACTION_MANAGEMENT.md)

FastAPI 프로젝트의 테스트 전략입니다. **Testcontainers 기반 API 테스트**와 **Mock 기반 Unit 테스트**로 계층별 테스트를 작성합니다.

---

## 1. 테스트 계층 구조

```
tests/
├── unit/                           # Mock 기반 단위 테스트 (빠름)
│   ├── domain/
│   │   ├── models/
│   │   │   └── test_user.py       # 도메인 모델 로직 테스트
│   │   └── services/
│   │       └── test_user_domain_service.py
│   ├── application/
│   │   └── services/
│   │       ├── test_user_app_service.py
│   │       └── test_user_query_service.py
│   └── infra/
│       └── repositories/
│           └── test_user_repository.py
├── api/                            # Testcontainers 기반 API 통합 테스트 (느림)
│   ├── {module}/
│   │   ├── conftest.py            # 모듈별 PopulateHelper
│   │   └── interface/
│   │       └── routers/
│   │           └── test_user_router.py
│   └── repositories/
│       └── test_user_repository_integration.py
└── conftest.py                     # Pytest fixtures (전역)
```

---

## 2. 테스트 작성 순서

### 개발 단계별 테스트 전략

1. **Domain Layer 구현** → Domain Model/Service 단위 테스트
   - 순수 비즈니스 로직 검증
   - Mock 불필요 (외부 의존성 없음)

2. **Application Layer 구현** → AppService/IntService/QueryService 단위 테스트
   - Repository/Transaction Mock으로 대체
   - 비즈니스 오케스트레이션 검증

3. **Infrastructure Layer 구현** → 단위 테스트 작성하지 않음
   - Repository 구현체는 API 통합 테스트로 검증
   - Session Mock 테스트는 선택적

4. **Interface Layer 구현** → 단위 테스트 작성하지 않음
   - Router/Schema는 API 통합 테스트로 검증

5. **API 통합 테스트 작성** → 전체 플로우 검증
   - Testcontainers로 실제 DB 환경 사용
   - E2E 시나리오 검증

### 테스트 작성 우선순위

**높음 (필수)**:
- Domain Model 비즈니스 로직
- Application Service 오케스트레이션
- API 통합 테스트 (Critical Path)

**중간 (권장)**:
- Domain Service 복잡한 로직
- Repository 통합 테스트

**낮음 (선택)**:
- Repository 단위 테스트 (Session Mock)
- Schema 검증 테스트

---

## 3. Unit Test (Mock 기반)

### 3.1 원칙
- **의존성을 Mock으로 대체**하여 격리된 환경에서 테스트
- **빠른 실행 속도** (DB, 네트워크 없음)
- **단일 책임 검증** (클래스/메서드 단위)
- `pytest-mock` 또는 `unittest.mock` 사용

### 2.2 Domain Model 테스트

도메인 모델은 **순수 비즈니스 로직**만 검증. Mock 불필요 (의존성 없음).

```python
# tests/unit/domain/models/test_user.py
import pytest
from domain.models.user import User

class TestUser:
    """User 도메인 모델 테스트"""
    
    def test_create_user_success(self):
        """사용자 생성 성공"""
        # Given
        email = "test@example.com"
        name = "Test User"
        
        # When
        user = User.create(email=email, name=name)
        
        # Then
        assert user.email == email
        assert user.name == name
        assert user.user_id is not None
    
    def test_create_user_with_invalid_email(self):
        """잘못된 이메일로 생성 실패"""
        # Given
        invalid_email = "invalid-email"
        
        # When / Then
        with pytest.raises(ValueError, match="Invalid email format"):
            User.create(email=invalid_email, name="Test")
    
    def test_change_profile(self):
        """프로필 변경"""
        # Given
        user = User.create(email="old@example.com", name="Old Name")
        
        # When
        updated = user.change_profile(email="new@example.com", name="New Name")
        
        # Then
        assert updated.email == "new@example.com"
        assert updated.name == "New Name"
        assert updated.user_id == user.user_id  # ID는 불변
```

### 2.3 Application Service 테스트 (Mock Repository)

AppService는 **Repository를 Mock**으로 대체하여 비즈니스 오케스트레이션만 검증.

```python
# tests/unit/application/services/test_user_app_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from application.services.user_app_service import UserAppService
from application.dtos.user_dto import RegisterUserCommand, RegisterUserCommandResult
from domain.models.user import User
from domain.protocols.user_repository_protocol import UserRepositoryProtocol
from shared.protocols.transaction_protocol import TransactionProtocol

@pytest.fixture
def mock_user_repo():
    """Mock UserRepository"""
    repo = AsyncMock(spec=UserRepositoryProtocol)
    return repo

@pytest.fixture
def mock_transaction():
    """Mock Transaction"""
    transaction = AsyncMock(spec=TransactionProtocol)
    # Context manager mock
    transaction.__aenter__ = AsyncMock(return_value=transaction)
    transaction.__aexit__ = AsyncMock(return_value=None)
    return transaction

@pytest.fixture
def user_app_service(mock_user_repo, mock_transaction):
    """UserAppService 인스턴스"""
    return UserAppService(
        user_repo=mock_user_repo,
        transaction=mock_transaction
    )

class TestUserAppService:
    """UserAppService 단위 테스트"""
    
    @pytest.mark.asyncio
    async def test_register_user_success(
        self, 
        user_app_service, 
        mock_user_repo
    ):
        """사용자 등록 성공"""
        # Given
        command = RegisterUserCommand(
            email="test@example.com",
            name="Test User"
        )
        expected_user = User.create(
            email=command.email,
            name=command.name
        )
        mock_user_repo.save.return_value = expected_user
        
        # When
        result = await user_app_service.register_user(command)
        
        # Then
        assert isinstance(result, RegisterUserCommandResult)
        assert result.user_id == expected_user.user_id
        assert result.email == expected_user.email
        mock_user_repo.save.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_register_user_duplicate_email(
        self,
        user_app_service,
        mock_user_repo
    ):
        """중복 이메일로 등록 실패"""
        # Given
        command = RegisterUserCommand(
            email="duplicate@example.com",
            name="Test User"
        )
        mock_user_repo.exists_by_email.return_value = True
        
        # When / Then
        with pytest.raises(ValueError, match="Email already exists"):
            await user_app_service.register_user(command)
        
        mock_user_repo.exists_by_email.assert_called_once_with("duplicate@example.com")
        mock_user_repo.save.assert_not_called()
```

### 2.4 Repository 테스트 (Mock Session)

Repository 단위 테스트는 **SQLAlchemy Session을 Mock**으로 대체.

```python
# tests/unit/infra/repositories/test_user_repository.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from sqlalchemy.ext.asyncio import AsyncSession
from infra.repositories.user_repository import UserRepository
from infra.orm.entities.user_entity import UserEntity
from domain.models.user import User

@pytest.fixture
def mock_session():
    """Mock AsyncSession"""
    session = AsyncMock(spec=AsyncSession)
    return session

@pytest.fixture
def user_repository(mock_session):
    """UserRepository 인스턴스"""
    return UserRepository(session=mock_session)

class TestUserRepository:
    """UserRepository 단위 테스트"""
    
    @pytest.mark.asyncio
    async def test_save_user(self, user_repository, mock_session):
        """사용자 저장"""
        # Given
        user = User.create(email="test@example.com", name="Test User")
        
        # Mock entity after save
        saved_entity = UserEntity(
            user_id=user.user_id,
            email=user.email,
            name=user.name
        )
        mock_session.execute.return_value.scalar_one.return_value = saved_entity
        
        # When
        result = await user_repository.save(user)
        
        # Then
        assert result.user_id == user.user_id
        assert result.email == user.email
        mock_session.add.assert_called_once()
        mock_session.flush.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_get_by_id_not_found(self, user_repository, mock_session):
        """ID로 조회 실패"""
        # Given
        mock_session.execute.return_value.scalar_one_or_none.return_value = None
        
        # When
        result = await user_repository.get_by_id(999)
        
        # Then
        assert result is None
```

---

## 4. API Integration Test (Testcontainers 기반)

### 4.1 원칙
- **실제 인프라 사용** (PostgreSQL, Redis 등)
- **E2E 시나리오 검증** (API → Service → Repository → DB)
- **Testcontainers로 격리된 환경 보장**
- 느리지만 실제 동작 보장
### 4.2 테스트 클래스 분리 전략

#### HTTP 메서드별 클래스 분리 (권장)

API 테스트는 HTTP 메서드별로 독립된 클래스로 분리하여 테스트 격리와 가독성을 향상시킵니다.

**네이밍 규칙**:
- POST: `Test{Resource}RouterPost`
- GET: `Test{Resource}RouterGet`
- PUT: `Test{Resource}RouterPut`
- DELETE: `Test{Resource}RouterDelete`

**장점**:
- 테스트 독립성 보장
- 가독성 향상
- 특정 HTTP 메서드만 선택 실행 가능
- 각 메서드별 fixture 분리 용이

**예시**:
```python
# tests/api/user/interface/routers/test_user_router.py

class TestUserRouterPost:
    """POST /users 엔드포인트 테스트"""
    
    @pytest.mark.asyncio
    async def test_post_user_success(self, test_client):
        # POST 테스트
        pass
    
    @pytest.mark.asyncio
    async def test_post_user_duplicate_email(self, test_client):
        # 중복 이메일 에러 테스트
        pass


class TestUserRouterGet:
    """GET /users 엔드포인트 테스트"""
    
    @pytest.mark.asyncio
    async def test_get_user_success(self, test_client):
        # GET 단건 조회
        pass
    
    @pytest.mark.asyncio
    async def test_get_users_list(self, test_client):
        # GET 목록 조회
        pass


class TestUserRouterPut:
    """PUT /users/{id} 엔드포인트 테스트"""
    
    @pytest.fixture
    async def put_test_data(self, populate_helper):
        """PUT 테스트용 데이터"""
        # PUT 전용 fixture
        pass
    
    @pytest.mark.asyncio
    async def test_put_user_success(self, test_client, put_test_data):
        # PUT 테스트
        pass


class TestUserRouterDelete:
    """DELETE /users/{id} 엔드포인트 테스트"""
    
    @pytest.fixture
    async def delete_test_data(self, populate_helper):
        """DELETE 테스트용 데이터"""
        # DELETE 전용 fixture
        pass
    
    @pytest.mark.asyncio
    async def test_delete_user_success(self, test_client, delete_test_data):
        # DELETE 테스트
        pass
```

**실행 예시**:
```bash
# 특정 클래스만 실행
pytest tests/api/user/interface/routers/test_user_router.py::TestUserRouterPost -v

# 특정 메서드만 실행
pytest tests/api/user/interface/routers/test_user_router.py::TestUserRouterPost::test_post_user_success -v
```
### 4.3 PopulateHelper 패턴 (API 테스트 데이터 관리)

#### 개요

API 통합 테스트에서 테스트 데이터 준비 및 정리를 일관되게 관리하기 위한 헬퍼 클래스 패턴입니다.

**특징**:
- DB 직접 조작으로 빠른 데이터 준비
- ID 충돌 방지 메커니즘 내장
- 모듈별 conftest.py에 정의
- 키워드 인자 강제로 명확성 확보

#### PopulateHelper 구조

```python
# tests/api/{module}/conftest.py
import random
from sqlalchemy.ext.asyncio import AsyncSession

class PopulateHelper:
    """모듈별 테스트 데이터 관리 헬퍼"""
    
    def __init__(self, session: AsyncSession):
        self.session = session
        # ID 충돌 방지를 위한 랜덤 베이스
        self.ID_BASE = random.randint(100, 999) * 1000
    
    async def insert_user(self, *, user_id: int, email: str, name: str):
        """사용자 생성 (키워드 인자 강제)"""
        user_entity = UserEntity(
            user_id=user_id,
            email=email,
            name=name
        )
        self.session.add(user_entity)
        await self.session.flush()
    
    async def delete_user(self, *, user_id: int):
        """사용자 삭제"""
        await self.session.execute(
            delete(UserEntity).where(UserEntity.user_id == user_id)
        )
        await self.session.flush()
    
    async def get_user(self, *, user_id: int):
        """사용자 조회"""
        result = await self.session.execute(
            select(UserEntity).where(UserEntity.user_id == user_id)
        )
        return result.scalar_one_or_none()


@pytest.fixture(scope="package")
async def populate_helper(test_session):
    """PopulateHelper 인스턴스"""
    return PopulateHelper(session=test_session)
```

#### 테스트 데이터 ID 규칙

**ID 충돌 방지 전략**:
- `ID_BASE = random.randint(100, 999) * 1000` (100000 ~ 999000)
- Package 공통 데이터: `-1000 - ID_BASE`, `-2000 - ID_BASE`
- Function별 데이터: `-9999 - ID_BASE`, `-9998 - ID_BASE`

**예시**:
```python
# Package scope fixture (모듈 전체 공통)
@pytest.fixture(scope="package")
async def populate_db(populate_helper):
    """모듈 공통 테스트 데이터"""
    company_id = -1000 - populate_helper.ID_BASE
    user_id = -2000 - populate_helper.ID_BASE
    
    try:
        await populate_helper.insert_company(company_id=company_id, ...)
        await populate_helper.insert_user(user_id=user_id, ...)
        
        yield {"company_id": company_id, "user_id": user_id}
    finally:
        # 역순 삭제 (외래키 제약조건)
        await populate_helper.delete_user(user_id=user_id)
        await populate_helper.delete_company(company_id=company_id)


# Function scope fixture (테스트별 독립)
@pytest.fixture
async def put_test_data(populate_helper):
    """PUT 테스트용 데이터"""
    user_id = -9999 - populate_helper.ID_BASE
    
    try:
        await populate_helper.insert_user(
            user_id=user_id,
            email=f"put_test_{populate_helper.ID_BASE}@example.com",
            name="Put Test User"
        )
        yield user_id
    finally:
        try:
            await populate_helper.delete_user(user_id=user_id)
        except Exception:
            pass  # 이미 삭제되었을 수 있음
```

#### 사용 예시

```python
class TestUserRouterPut:
    @pytest.fixture
    async def put_test_data(self, populate_helper):
        user_id = -9999 - populate_helper.ID_BASE
        try:
            await populate_helper.insert_user(
                user_id=user_id,
                email="test@example.com",
                name="Test User"
            )
            yield user_id
        finally:
            try:
                await populate_helper.delete_user(user_id=user_id)
            except Exception:
                pass
    
    @pytest.mark.asyncio
    async def test_put_user_success(
        self, test_client, populate_helper, put_test_data
    ):
        # Given
        user_id = put_test_data
        update_data = {
            "email": "updated@example.com",
            "name": "Updated Name"
        }
        
        # When
        response = await test_client.put(
            f"/api/users/{user_id}",
            json=update_data
        )
        
        # Then
        assert response.status_code == 200
        
        # DB 검증
        db_user = await populate_helper.get_user(user_id=user_id)
        assert db_user.email == update_data["email"]
        assert db_user.name == update_data["name"]
```

---

### 4.4 Testcontainers 설정

```python
# tests/conftest.py
import pytest
import asyncio
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from httpx import AsyncClient
from main import app  # FastAPI app
from infra.orm.base import Base

@pytest.fixture(scope="session")
def event_loop():
    """세션 스코프 이벤트 루프"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session")
def postgres_container():
    """PostgreSQL Testcontainer"""
    with PostgresContainer("postgres:15-alpine") as postgres:
        yield postgres

@pytest.fixture(scope="session")
async def test_engine(postgres_container):
    """테스트용 SQLAlchemy Engine"""
    database_url = postgres_container.get_connection_url().replace(
        "psycopg2", "asyncpg"
    )
    engine = create_async_engine(database_url, echo=True)
    
    # 테이블 생성
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    yield engine
    
    # 정리
    await engine.dispose()

@pytest.fixture
async def test_session(test_engine):
    """테스트용 AsyncSession"""
    async_session = sessionmaker(
        test_engine, 
        class_=AsyncSession, 
        expire_on_commit=False
    )
    
    async with async_session() as session:
        yield session
        await session.rollback()  # 테스트 후 롤백

@pytest.fixture
async def test_client(test_session):
    """테스트용 FastAPI Client"""
    # DI 오버라이드
    from core.dependencies import get_session
    
    async def override_get_session():
        yield test_session
    
    app.dependency_overrides[get_session] = override_get_session
    
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client
    
    app.dependency_overrides.clear()
```

### 4.5 API Integration Test

```python
# tests/integration/api/test_user_api.py
import pytest
from httpx import AsyncClient

class TestUserApi:
    """User API 통합 테스트 (E2E)"""
    
    @pytest.mark.asyncio
    async def test_create_user_success(self, test_client: AsyncClient):
        """사용자 생성 성공 (E2E)"""
        # Given
        request_data = {
            "email": "test@example.com",
            "name": "Test User"
        }
        
        # When
        response = await test_client.post("/api/users", json=request_data)
        
        # Then
        assert response.status_code == 201
        data = response.json()
        assert data["code"] == "CREATED"
        assert data["message"] == "User created successfully"
        assert data["data"]["email"] == "test@example.com"
        assert data["data"]["id"] is not None
    
    @pytest.mark.asyncio
    async def test_create_user_duplicate_email(self, test_client: AsyncClient):
        """중복 이메일로 생성 실패"""
        # Given
        request_data = {
            "email": "duplicate@example.com",
            "name": "Test User"
        }
        
        # 첫 번째 생성 성공
        await test_client.post("/api/users", json=request_data)
        
        # When - 동일 이메일로 재시도
        response = await test_client.post("/api/users", json=request_data)
        
        # Then
        assert response.status_code == 400
        data = response.json()
        assert data["code"] == "USER_EMAIL_DUPLICATED"
        assert "already exists" in data["message"].lower()
    
    @pytest.mark.asyncio
    async def test_get_user_list(self, test_client: AsyncClient):
        """사용자 목록 조회"""
        # Given - 여러 사용자 생성
        for i in range(3):
            await test_client.post("/api/users", json={
                "email": f"user{i}@example.com",
                "name": f"User {i}"
            })
        
        # When
        response = await test_client.get("/api/users")
        
        # Then
        assert response.status_code == 200
        data = response.json()
        assert data["code"] == "SUCCESS"
        assert len(data["data"]["items"]) >= 3
    
    @pytest.mark.asyncio
    async def test_update_user_success(self, test_client: AsyncClient):
        """사용자 수정 성공"""
        # Given - 사용자 생성
        create_response = await test_client.post("/api/users", json={
            "email": "old@example.com",
            "name": "Old Name"
        })
        user_id = create_response.json()["data"]["id"]
        
        # When - 수정
        update_data = {
            "email": "new@example.com",
            "name": "New Name"
        }
        response = await test_client.put(f"/api/users/{user_id}", json=update_data)
        
        # Then
        assert response.status_code == 200
        data = response.json()
        assert data["code"] == "UPDATED"
        assert data["data"]["email"] == "new@example.com"
        assert data["data"]["name"] == "New Name"
    
    @pytest.mark.asyncio
    async def test_delete_user_success(self, test_client: AsyncClient):
        """사용자 삭제 성공"""
        # Given - 사용자 생성
        create_response = await test_client.post("/api/users", json={
            "email": "delete@example.com",
            "name": "Delete User"
        })
        user_id = create_response.json()["data"]["id"]
        
        # When - 삭제
        response = await test_client.delete(f"/api/users/{user_id}")
        
        # Then
        assert response.status_code == 200
        data = response.json()
        assert data["code"] == "DELETED"
        
        # 삭제 확인
        get_response = await test_client.get(f"/api/users/{user_id}")
        assert get_response.status_code == 404
```

### 4.6 Repository Integration Test

Repository 통합 테스트는 **실제 DB**와 상호작용을 검증합니다.

```python
# tests/integration/repositories/test_user_repository_integration.py
import pytest
from infra.repositories.user_repository import UserRepository
from domain.models.user import User

class TestUserRepositoryIntegration:
    """UserRepository 통합 테스트 (실제 DB)"""
    
    @pytest.mark.asyncio
    async def test_save_and_retrieve(self, test_session):
        """저장 후 조회"""
        # Given
        repo = UserRepository(session=test_session)
        user = User.create(email="test@example.com", name="Test User")
        
        # When - 저장
        saved = await repo.save(user)
        await test_session.commit()
        
        # Then - 조회
        retrieved = await repo.get_by_id(saved.user_id)
        assert retrieved is not None
        assert retrieved.email == user.email
        assert retrieved.name == user.name
    
    @pytest.mark.asyncio
    async def test_exists_by_email(self, test_session):
        """이메일로 존재 확인"""
        # Given
        repo = UserRepository(session=test_session)
        user = User.create(email="exists@example.com", name="Test")
        await repo.save(user)
        await test_session.commit()
        
        # When / Then
        assert await repo.exists_by_email("exists@example.com") is True
        assert await repo.exists_by_email("notexists@example.com") is False
    
    @pytest.mark.asyncio
    async def test_query_users_with_criteria(self, test_session):
        """Criteria로 사용자 목록 조회"""
        # Given
        repo = UserRepository(session=test_session)
        for i in range(5):
            user = User.create(email=f"user{i}@example.com", name=f"User {i}")
            await repo.save(user)
        await test_session.commit()
        
        # When
        from domain.models.criteria.user_paged_criteria import UserPagedCriteria
        criteria = UserPagedCriteria(page=1, page_size=3)
        result = await repo.query(criteria)
        
        # Then
        assert len(result.items) == 3
        assert result.total_count >= 5
```

---

### 4.7 외래키 제약조건 고려

#### 데이터 생성/삭제 순서

**생성 순서**: 부모 테이블 → 자식 테이블
```python
@pytest.fixture(scope="package")
async def populate_db(populate_helper):
    try:
        # 1. 부모 테이블 먼저 생성
        await populate_helper.insert_company(company_id=-1000, ...)
        
        # 2. 자식 테이블 생성 (company_id 참조)
        await populate_helper.insert_user(
            user_id=-2000,
            company_id=-1000,  # FK 참조
            ...
        )
        
        yield {"company_id": -1000, "user_id": -2000}
    finally:
        # 역순 삭제
        pass
```

**삭제 순서**: 자식 테이블 → 부모 테이블 (역순)
```python
    finally:
        # 1. 자식 먼저 삭제
        await populate_helper.delete_user(user_id=-2000)
        
        # 2. 부모 삭제
        await populate_helper.delete_company(company_id=-1000)
```

#### try-finally 필수

테스트 실패 시에도 데이터 정리를 보장하기 위해 try-finally 패턴을 반드시 사용합니다.

```python
@pytest.fixture
async def test_data(populate_helper):
    user_id = -9999 - populate_helper.ID_BASE
    order_id = -9998 - populate_helper.ID_BASE
    
    try:
        # 데이터 생성
        await populate_helper.insert_user(user_id=user_id, ...)
        await populate_helper.insert_order(
            order_id=order_id,
            user_id=user_id,  # FK
            ...
        )
        
        yield {"user_id": user_id, "order_id": order_id}
    
    finally:
        # 역순 삭제 (테스트 실패해도 실행)
        try:
            await populate_helper.delete_order(order_id=order_id)
        except Exception:
            pass  # 이미 삭제되었을 수 있음
        
        try:
            await populate_helper.delete_user(user_id=user_id)
        except Exception:
            pass
```

#### 외래키 ON DELETE CASCADE 활용

스키마에서 `ON DELETE CASCADE` 설정 시 부모 삭제만으로 자식도 자동 삭제됩니다.

```sql
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_user_id
    FOREIGN KEY (user_id)
    REFERENCES users(user_id)
    ON DELETE CASCADE;  -- 부모 삭제 시 자식도 자동 삭제
```

```python
finally:
    # CASCADE 설정 시 user 삭제만으로 order도 삭제됨
    await populate_helper.delete_user(user_id=user_id)
```

---

## 5. 테스트 실행 전략

### 5.1 테스트 분리 실행

```bash
# Unit 테스트만 실행 (빠름)
pytest tests/unit/ -v

# Integration 테스트만 실행 (느림)
pytest tests/integration/ -v

# 전체 테스트
pytest tests/ -v

# 특정 파일만
pytest tests/unit/application/services/test_user_app_service.py -v

# 커버리지 포함
pytest tests/ --cov=src --cov-report=html
```

### 5.2 Pytest Markers

```python
# tests/conftest.py
import pytest

def pytest_configure(config):
    config.addinivalue_line("markers", "unit: Unit tests with mocks")
    config.addinivalue_line("markers", "integration: Integration tests with real infra")
    config.addinivalue_line("markers", "slow: Slow running tests")
```

```python
# 테스트에 마커 추가
@pytest.mark.unit
def test_something():
    pass

@pytest.mark.integration
@pytest.mark.slow
async def test_api():
    pass
```

```bash
# 마커로 필터링
pytest -m unit          # Unit 테스트만
pytest -m integration   # Integration 테스트만
pytest -m "not slow"    # 느린 테스트 제외
```

### 5.3 CI/CD에서 실행

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-asyncio pytest-cov pytest-mock
      - name: Run unit tests
        run: pytest tests/unit/ -v --cov=src
  
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-asyncio testcontainers
      - name: Run integration tests
        run: pytest tests/integration/ -v
```

---

### 5.4 코드 포맷팅 체크리스트

#### 테스트 작성 완료 후 필수 실행

**명령어**:
```bash
# 단위 테스트 포맷팅
black tests/unit/

# API 테스트 포맷팅
black tests/api/

# 전체 테스트 포맷팅
black tests/

# 소스코드 포맷팅
black src/
```

**실행 시점**:
- [ ] 새로운 단위 테스트 작성 후
- [ ] 새로운 API 테스트 작성 후
- [ ] 기존 테스트 수정 후
- [ ] PR/커밋 전 최종 확인
- [ ] CI/CD 파이프라인에서 자동 검증

**목적**:
- 일관된 코드 스타일 유지
- 가독성 향상 및 유지보수성 증대
- 코드 리뷰 시 포맷팅 이슈 방지
- 팀 전체 코딩 컨벤션 통일

**CI/CD 통합**:
```yaml
# .github/workflows/test.yml
- name: Check code formatting
  run: |
    black --check tests/
    black --check src/

- name: Run tests
  run: pytest tests/ -v
```

---

## 6. 테스트 작성 가이드

### 6.1 언제 Mock을 사용할까?

| 레이어 | Mock 여부 | 이유 |
|--------|-----------|------|
| Domain Model | ❌ Mock 불필요 | 순수 로직, 외부 의존성 없음 |
| Domain Service | ⚠️ 선택적 | Repository만 Mock, 도메인 로직 검증 |
| App Service | ✅ Mock 필수 | Repository/Transaction Mock으로 오케스트레이션 검증 |
| Repository (단위) | ✅ Mock 필수 | Session Mock으로 매핑 로직만 검증 |
| Repository (통합) | ❌ 실제 DB | Testcontainers로 실제 쿼리 검증 |
| API (E2E) | ❌ 실제 인프라 | Testcontainers로 전체 흐름 검증 |

### 6.2 테스트 네이밍 컨벤션

```python
# 패턴: test_{method}_{scenario}
def test_register_user_success():
    pass

def test_register_user_duplicate_email():
    pass

def test_get_user_by_id_not_found():
    pass

# Given-When-Then 주석 사용
def test_example():
    # Given (준비)
    user = User.create(...)
    
    # When (실행)
    result = service.do_something(user)
    
    # Then (검증)
    assert result.success is True
```

### 6.3 Fixture 재사용

```python
# tests/conftest.py - 공통 fixture
@pytest.fixture
def sample_user():
    """테스트용 샘플 사용자"""
    return User.create(
        email="sample@example.com",
        name="Sample User"
    )

# tests/unit/conftest.py - Unit 테스트 전용
@pytest.fixture
def mock_user_repo():
    return AsyncMock(spec=UserRepositoryProtocol)

# tests/integration/conftest.py - Integration 테스트 전용
@pytest.fixture
async def test_session(test_engine):
    # ... Testcontainer session
```

---

## 7. 의존성 설치

```txt
# requirements-test.txt
pytest>=7.4.0
pytest-asyncio>=0.21.0
pytest-cov>=4.1.0
pytest-mock>=3.11.0
testcontainers>=3.7.0
httpx>=0.24.0  # FastAPI 테스트용
```

```bash
pip install -r requirements-test.txt
```

---

## 8. 요약

### 테스트 작성 순서
1. Domain → Application → Infrastructure → Interface
2. 각 단계마다 해당 레이어 테스트 작성
3. 마지막에 API 통합 테스트로 전체 검증

### Unit Test (Mock)
- **빠른 피드백**, 개발 중 자주 실행
- Repository/Transaction을 Mock으로 대체
- 비즈니스 로직/오케스트레이션 검증

### API Integration Test (Testcontainers)
- **실제 동작 보장**, CI/CD에서 실행
- 실제 DB/인프라 사용
- E2E 시나리오 검증
- HTTP 메서드별 클래스 분리
- PopulateHelper로 테스트 데이터 관리

### 권장 비율
- Unit : API Integration = **70:30** 또는 **80:20**
- Unit 테스트로 대부분 커버, API Integration으로 실제 동작 검증
- Critical path는 반드시 API Integration 테스트 작성

### 필수 체크리스트
- [ ] 테스트 작성 순서 준수 (Domain → Application → API)
- [ ] HTTP 메서드별 테스트 클래스 분리
- [ ] PopulateHelper로 테스트 데이터 관리
- [ ] ID_BASE로 ID 충돌 방지
- [ ] 외래키 제약조건 고려 (생성/삭제 순서)
- [ ] try-finally로 테스트 데이터 정리 보장
- [ ] 테스트 완료 후 black 포맷팅 실행

---

## 버전 이력
- 1.1.0 (2026-01-11): 회사 베스트 프랙티스 적용 (테스트 작성 순서, HTTP 메서드별 클래스 분리, PopulateHelper 패턴, ID 충돌 방지, 외래키 제약조건, 코드 포맷팅 체크리스트)
- 1.0.0 (2026-01-11): 초기 테스트 전략 문서 작성 (Mock 기반 Unit / Testcontainers 기반 Integration)
