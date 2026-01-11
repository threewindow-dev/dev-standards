# Python 네이밍 컨벤션 (Naming Convention)

## 개요
- Python 프로젝트에서 일관된 네이밍 규칙을 적용하여 코드 가독성과 유지보수성을 향상시킵니다.
- PEP 8 기준을 따르며, DDD/Clean Architecture 구조에 맞는 추가 규칙을 포함합니다.

---

## 1. 모듈(파일) 네이밍

### 기본 규칙
- **소문자 + 언더스코어** (snake_case) 사용
# Python 네이밍 컨벤션 (Naming Convention)

이 문서는 가독성을 위해 주제별 파일로 분리되었습니다. 필요한 섹션을 아래 링크에서 확인하세요.

## 인덱스
- 모듈(파일) 네이밍: [naming_modules.md](naming_modules.md)
- Schema & DTO 네이밍: [naming_schema_dto.md](naming_schema_dto.md)
- Repository Model/Entity 네이밍: [naming_repository_model.md](naming_repository_model.md)
- Domain Model 네이밍: [naming_domain_model.md](naming_domain_model.md)
- 요약 & 변경 이력: [naming_summary.md](naming_summary.md)

## 참고
- 기존 요약 표와 버전 이력은 [naming_summary.md](naming_summary.md)에서 확인하세요.
        result = await self.db.execute(query)
        
        # DB 결과를 다시 Entity로 변환하여 반환
        return UserModel(
            user_id=result.lastrowid,
            email=entity.email,
            name=entity.name,
            created_at=datetime.now(),
            updated_at=datetime.now()
        )
```

---

#### Entity와 DTO 분리의 장점

##### 1. 계층 간 독립적 진화 (Independent Evolution)
```python
# 시나리오: DB 스키마 변경 필요
# - users 테이블에 "profile_image_url" 컬럼 추가
# - 하지만 API는 여전히 이전 필드만 응답해야 함

# Entity 계층 (DB)
class UserModel(BaseModel):
    user_id: int
    email: str
    name: str
    profile_image_url: str  # 새로 추가됨
    created_at: datetime

# DTO 계층 (App/API) - 변경 없음
class RegisterUserCommandResult(BaseModel):
    user_id: int
    created_at: datetime

# Repository에서 변환 로직만 관리
def map_entity_to_dto(entity: UserModel) -> RegisterUserCommandResult:
    # profile_image_url은 무시하고 필요한 필드만 반환
    return RegisterUserCommandResult(
        user_id=entity.user_id,
        created_at=entity.created_at
    )
```
**장점**: DB 스키마 변경이 API에 영향을 주지 않음. 클라이언트가 마이그레이션할 필요 없음.

##### 2. 보안 및 데이터 노출 제어 (Data Exposure Control)
```python
# 시나리오: User 조회 후 민감한 필드를 숨겨야 함

# Entity (DB에서 조회된 모든 데이터)
class UserModel(BaseModel):
    user_id: int
    email: str
    name: str
    password_hash: str  # 민감함
    ssn: str  # 민감함
    stripe_api_key: str  # 민감함
    created_at: datetime

# DTO (API 응답에는 포함되지 않음)
class UserDto(BaseModel):
    user_id: int
    email: str
    name: str
    created_at: datetime
    # password_hash, ssn, stripe_api_key는 절대 여기 없음

# Repository → AppService 변환
async def get_user(self, user_id: int) -> UserDto:
    entity = await self.user_repo.get_by_id(user_id)
    # Entity에는 모든 필드가 있지만, DTO는 공개해도 되는 필드만 포함
    return UserDto(
        user_id=entity.user_id,
        email=entity.email,
        name=entity.name,
        created_at=entity.created_at
    )
```
**장점**: 민감한 데이터를 실수로 노출할 위험을 줄임. 계층별 권한 관리가 명확함.

##### 3. 데이터 정규화/비정규화 자유도 (Flexibility in DB Design)
```python
# 시나리오: 조회 성능을 위해 DB에서 비정규화된 데이터 저장
# users 테이블에 company_name (join 없이 바로 조회하기 위함)

# Entity (DB에서 실제 저장된 구조)
class UserModel(BaseModel):
    user_id: int
    email: str
    name: str
    company_id: int
    company_name: str  # 비정규화됨 (companies 테이블과 중복)
    created_at: datetime

# DTO (앱 로직은 정규화된 구조로 처리)
class UserDto(BaseModel):
    user_id: int
    email: str
    name: str
    company_id: int
    # company_name은 별도로 Company.name으로 처리
    created_at: datetime

# Repository에서 정규화/비정규화 처리
async def get_user(self, user_id: int) -> UserDto:
    entity = await self.user_repo.get_by_id(user_id)
    # Entity의 company_name은 무시하고, company_id만 DTO에 포함
    return UserDto(
        user_id=entity.user_id,
        email=entity.email,
        name=entity.name,
        company_id=entity.company_id,
        created_at=entity.created_at
    )
```
**장점**: DB 성능 최적화(비정규화, 캐싱 등)를 자유롭게 할 수 있음. 앱 로직은 정규화된 깔끔한 구조 유지.

##### 4. ORM 활용 (ORM Mapping Benefits)
```python
# 시나리오: SQLAlchemy ORM 사용

# Entity - SQLAlchemy 모델 (DB 스키마와 1:1 매핑)
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import relationship

class UserEntity(Base):
    __tablename__ = "users"
    
    user_id = Column(Integer, primary_key=True)
    email = Column(String)
    name = Column(String)
    password_hash = Column(String)
    created_at = Column(DateTime)
    
    # Relationship (자동 로드)
    orders = relationship("OrderEntity", back_populates="user")

# DTO - 간단한 Pydantic 모델 (데이터 전달용)
class UserDto(BaseModel):
    user_id: int
    email: str
    name: str
    created_at: datetime

# Repository
class UserRepository:
    async def get_by_id(self, user_id: int) -> UserEntity:
        # ORM의 강력한 기능 활용: 자동 relationship 로드, 캐싱 등
        return await session.query(UserEntity).filter_by(user_id=user_id).first()
    
    async def get_user_dto(self, user_id: int) -> UserDto:
        entity = await self.get_by_id(user_id)
        # Entity → DTO 변환
        return UserDto(
            user_id=entity.user_id,
            email=entity.email,
            name=entity.name,
            created_at=entity.created_at
        )
```
**장점**: ORM의 lazy loading, relationship, 캐싱 등 고급 기능을 활용 가능. DTO는 단순하게 유지.

##### 5. 테스트 용이성 (Testability)
```python
# 시나리오: Repository 로직 테스트

# Mock Entity 생성이 간단함 (필요한 필드만)
mock_entity = UserModel(
    user_id=1,
    email="test@example.com",
    name="Test User",
    created_at=datetime.now()
)

# 실제 비즈니스 로직은 DTO로 테스트
# (Entity 세부사항에 의존하지 않음)

async def test_register_user():
    command = RegisterUserCommand(
        email="test@example.com",
        name="Test User",
        password="password123"
    )
    result = await service.register_user(command)
    assert result.user_id == 1
    # Entity 구조 변경이 이 테스트에 영향을 주지 않음
```
**장점**: 각 계층을 독립적으로 테스트 가능. 세부 구현 변경이 테스트에 미치는 영향 최소화.

##### 6. 마이그레이션 시 영향도 최소화 (Migration Safety)
```python
# 시나리오: 새로운 필드 추가 또는 필드 제거

# 구 버전: users_v1 테이블
class UserModelV1(BaseModel):
    user_id: int
    email: str
    name: str

# 신 버전: users_v2 테이블 (새 필드 추가)
class UserModelV2(BaseModel):
    user_id: int
    email: str
    name: str
    phone: str  # 신규
    
# Repository에서 버전 처리
class UserRepository:
    async def get_user_dto(self, user_id: int) -> UserDto:
        # v1 또는 v2 중 어느 것을 조회했든
        # DTO는 동일한 구조를 반환
        entity = await self._get_entity(user_id)
        return UserDto(
            user_id=entity.user_id,
            email=entity.email,
            name=entity.name
        )
```
**장점**: DB 마이그레이션 시 API 응답 구조를 유지 가능. 클라이언트 변경 최소화.

---

---

## 2.4 Domain Model 네이밍 규칙 (Optional: DDD 엄격 적용)

**상황**: Domain-Driven Design을 엄격하게 따르거나, 복잡한 비즈니스 로직을 표현해야 할 때, Domain Model을 별도로 정의할 수 있습니다. Domain Model은 **비즈니스 규칙을 캡슐화**하고 DB 영속성과는 완전히 독립적입니다.

### 개념 구분

| 개념 | 위치 | 책임 | 예시 |
|------|------|------|------|
| **Domain Model** | domain/models/ | 비즈니스 규칙, 비즈니스 제약 | `User`는 이메일 유효성, 암호 정책 검증 |
| **DTO** | application/dtos/ | 계층 간 데이터 전달 | `RegisterUserCommand` |
| **Entity** | infra/models/ | DB 스키마 매핑 | SQLAlchemy ORM 모델 |
| **Schema** | interface/schemas/ | HTTP 요청/응답 검증 | `PostUserRequest` |

### 네이밍 규칙

#### 기본 원칙
- **PascalCase** 사용
- **Resource는 단수형** (User, Order, Payment)
- **비즈니스 로직을 메서드로 표현** (명사가 아닌 동사 중심)
- **불변성(Immutability) 고려** - 상태 변경은 새로운 인스턴스 반환

#### 생성 (Factory 패턴)
- 정적 메서드로 도메인 모델 생성
- 패턴: `User.create()`, `User.register()`
- 예시:
```python
# domain/models/user.py
class User:
    def __init__(self, user_id: int, email: str, name: str, password_hash: str):
        self._user_id = user_id
        self._email = email
        self._name = name
        self._password_hash = password_hash
    
    @staticmethod
    def create(email: str, name: str, password: str) -> 'User':
        """비즈니스 규칙을 포함한 User 생성"""
        # 비즈니스 검증
        if not User.is_valid_email(email):
            raise InvalidEmailError(email)
        if not User.is_valid_password(password):
            raise WeakPasswordError("최소 8글자, 대소문자 포함 필요")
        
        # 비즈니스 규칙: 암호는 항상 해시됨
        hashed_pw = hash_password(password)
        
        # Domain Model은 ID가 없음 (DB 저장 전)
        return User(user_id=None, email=email, name=name, password_hash=hashed_pw)
    
    @staticmethod
    def is_valid_email(email: str) -> bool:
        """도메인 규칙: 이메일 유효성"""
        return "@" in email and "." in email.split("@")[1]
    
    @staticmethod
    def is_valid_password(password: str) -> bool:
        """도메인 규칙: 암호 정책"""
        return len(password) >= 8 and any(c.isupper() for c in password)
```

#### 상태 변경 (Command 메서드)
- 도메인 모델의 상태를 변경하는 메서드
- 패턴: `user.change_email()`, `user.update_name()`, `user.deactivate()`
- 예시:
```python
class User:
    def change_email(self, new_email: str) -> 'User':
        """이메일 변경 (새 인스턴스 반환 - 불변성)"""
        if not User.is_valid_email(new_email):
            raise InvalidEmailError(new_email)
        # 새로운 User 인스턴스 반환
        return User(
            user_id=self._user_id,
            email=new_email,
            name=self._name,
            password_hash=self._password_hash
        )
    
    def deactivate(self) -> 'User':
        """사용자 비활성화 (논리적 삭제)"""
        # 비활성화 상태를 나타내기 위해 name을 변경하거나 별도 필드 추가
        return User(
            user_id=self._user_id,
            email=self._email,
            name=self._name,
            password_hash=self._password_hash,
            is_active=False
        )
```

#### 조회 (Query 메서드)
- 도메인 상태를 조회하는 메서드
- 패턴: `user.get_email()`, `user.has_permission()`, `user.is_valid()`
- 예시:
```python
class User:
    def get_email(self) -> str:
        """이메일 조회 (캡슐화)"""
        return self._email
    
    def has_permission(self, permission: str) -> bool:
        """권한 확인 (도메인 규칙)"""
        return permission in self._permissions
    
    def can_delete_order(self, order_id: int) -> bool:
        """주문 삭제 가능 여부 (복잡한 도메인 규칙)"""
        # 예: 소유자이면서, 배송 전이어야 함
        return self._owns_order(order_id) and not self._is_shipped(order_id)
```

### 데이터 흐름 (Domain Model + Entity 함께 사용)

```
1. HTTP 요청
   ↓
2. PostUserRequest (Schema) 검증
   ↓
3. Router에서 DTO로 변환: RegisterUserCommand
   ↓
4. AppService에서 Domain Model 생성: User.create(email, name, password)
   (비즈니스 규칙 적용)
   ↓
5. Domain Model을 Entity로 변환
   ↓
6. Repository에 Entity 저장: DB insert
   ↓
7. Repository에서 생성된 ID를 포함한 Entity 반환
   ↓
8. Entity를 DTO로 변환: RegisterUserCommandResult
   ↓
9. PostUserResponse (Schema) 변환하여 HTTP 응답
```

**코드 예시**:
```python
# domain/models/user.py
class User:
    """도메인 모델 - 비즈니스 로직만 포함"""
    def __init__(self, user_id: int | None, email: str, name: str, password_hash: str):
        self._user_id = user_id
        self._email = email
        self._name = name
        self._password_hash = password_hash
    
    @staticmethod
    def create(email: str, name: str, password: str) -> 'User':
        # 비즈니스 검증
        if not User.is_valid_email(email):
            raise InvalidEmailError(email)
        hashed_pw = hash_password(password)
        return User(user_id=None, email=email, name=name, password_hash=hashed_pw)
    
    @property
    def email(self) -> str:
        return self._email
    
    @property
    def name(self) -> str:
        return self._name
    
    def get_password_hash(self) -> str:
        return self._password_hash

# infra/models/user_entity.py
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class UserEntity(Base):
    """DB Entity - 영속성 매핑만 담당"""
    __tablename__ = "users"
    
    user_id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    name = Column(String)
    password_hash = Column(String)
    created_at = Column(DateTime)
    updated_at = Column(DateTime)

# application/dtos/user_dto.py
class RegisterUserCommand(BaseModel):
    """DTO - 계층 간 전달"""
    email: str
    name: str
    password: str

class RegisterUserCommandResult(BaseModel):
    user_id: int
    created_at: datetime

# application/services/user_app_service.py
class UserAppService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo
    
    async def register_user(self, command: RegisterUserCommand) -> RegisterUserCommandResult:
        # 1. DTO → Domain Model (비즈니스 규칙 적용)
        domain_user = User.create(
            email=command.email,
            name=command.name,
            password=command.password
        )
        
        # 2. Domain Model → Entity
        entity = UserEntity(
            email=domain_user.email,
            name=domain_user.name,
            password_hash=domain_user.get_password_hash(),
            created_at=datetime.now(),
            updated_at=datetime.now()
        )
        
        # 3. Repository에 Entity 저장
        saved_entity = await self.user_repo.create(entity)
        
        # 4. Entity → DTO 변환
        return RegisterUserCommandResult(
            user_id=saved_entity.user_id,
            created_at=saved_entity.created_at
        )

# infra/repositories/user_repository.py
class UserRepository:
    async def create(self, entity: UserEntity) -> UserEntity:
        session.add(entity)
        await session.commit()
        await session.refresh(entity)
        return entity
```

---

### 계층 구조 비교

#### 단순 구조 (Schema + DTO)
```
HTTP Request (Schema)
        ↓
    DTO (Command/Dto)
        ↓
    Repository
        ↓
    Database
```
**용도**: 간단한 CRUD, 비즈니스 로직이 적음

#### 중간 구조 (Schema + DTO + Entity)
```
HTTP Request (Schema)
        ↓
    DTO (Command/Dto)
        ↓
    Repository
        ↓
   Entity (ORM)
        ↓
    Database
```
**용도**: 복잡한 DB 스키마, 성능 최적화 필요

#### 복잡 구조 (Schema + DTO + Domain Model + Entity)
```
HTTP Request (Schema)
        ↓
    DTO (Command/Dto)
        ↓
  Domain Model (비즈니스 로직)
        ↓
   Entity (ORM)
        ↓
    Repository
        ↓
    Database
```
**용도**: 복잡한 비즈니스 규칙, DDD 적용, 엔터프라이즈 프로젝트

---

### Domain Model vs Entity 분리의 장점

#### 1. 비즈니스 규칙 캡슐화
```python
# Domain Model에만 비즈니스 로직 포함
class Order:
    def can_cancel(self) -> bool:
        """도메인 규칙: 주문 취소 가능 여부"""
        return self.status == "pending" and self.created_at > (now() - timedelta(hours=24))
    
    def apply_discount(self, discount_rate: float) -> 'Order':
        """도메인 규칙: 할인율 검증"""
        if not (0 < discount_rate < 0.5):
            raise InvalidDiscountError("할인율은 0~50% 사이여야 함")
        return Order(..., discount_rate=discount_rate)

# Entity는 순수 데이터 저장소
class OrderEntity:
    order_id: int
    status: str
    discount_rate: float
    created_at: datetime
```

#### 2. DB 변경이 Domain 로직에 영향 없음
```python
# DB를 redesign해도 Domain Model은 유지
# 예: orders 테이블에 calculated_discount 캐싱 컬럼 추가

# Entity만 변경
class OrderEntity:
    order_id: int
    status: str
    discount_rate: float
    calculated_discount: float  # 캐싱용 (신규)
    created_at: datetime

# Domain Model은 변경 없음
class Order:
    def apply_discount(self, discount_rate: float) -> 'Order':
        # 동일한 비즈니스 로직
        if not (0 < discount_rate < 0.5):
            raise InvalidDiscountError(...)
        return Order(..., discount_rate=discount_rate)
```

#### 3. 테스트 독립성
```python
# Domain Model 테스트 (DB 없음, 빠름)
def test_order_can_cancel():
    order = Order(status="pending", created_at=datetime.now())
    assert order.can_cancel() == True

# Entity 테스트 (DB 포함, 별도)
async def test_order_entity_persistence():
    entity = OrderEntity(status="pending", ...)
    saved = await repo.create(entity)
    assert saved.order_id > 0
```

---

### 선택 기준

| 기준 | Schema + DTO만 | + Entity | + Domain Model |
|------|--------|---------|---------|
| **복잡도** | 낮음 | 중간 | 높음 |
| **학습곡선** | 낮음 | 중간 | 높음 |
| **비즈니스 로직** | 간단한 CRUD | 표준 CRUD | 복잡한 도메인 규칙 |
| **DB 의존도** | 높음 | 중간 | 낮음 (domain 우선) |
| **테스트 난이도** | 쉬움 | 중간 | 어려움 (대신 테스트 강화) |
| **적용 시점** | 프로젝트 초기 | 안정화 후 | 요구사항 복잡화 후 |
| **예시 프로젝트** | CRUD API, 스타트업 | 표준 웹앱 | 금융/결제/마켓플레이스 |

**권장사항**: 프로젝트 초기엔 **Schema + DTO**로 시작하고, 비즈니스 로직이 복잡해지면 **Entity**를 추가하고, 도메인 규칙이 극도로 복잡해지면 **Domain Model**을 도입하세요.



---

## 요약 테이블

| 레이어 | 접미사 | 단/복수 | 예시 |
|--------|--------|---------|------|
| Models (domain/models) | - (또는 `_model`) | 단수 | `user.py` |
| Criteria (domain/models/criteria) | `_criteria` | 단수 | `user_query_criteria.py` |
| ORM Entities (infra/orm/entities) | `_entity` (선택) | 단수 | `user_entity.py` |
| Protocols | `_protocol` | 단수 | `user_repository_protocol.py` |
| AppService | `_app_service` | 단수 | `user_app_service.py` |
| IntService | `_int_service` | 단수 | `payment_int_service.py` |
| DomainService | `_domain_service` | 단수 | `order_pricing_domain_service.py` |
| Repositories | `_repository` | 단수 | `user_repository.py` |
| DTOs | `_dto` | 단수 | `user_dto.py` |
| Schemas | `_schema` | 단수 | `user_schema.py` |
| Routers | `_router` | 단수 | `user_router.py` |
| Clients | `_client` | 단수 | `payment_client.py` |

---

## 버전 이력
- 1.7.2 (2026-01-11): UserCriteria와 UserPagedCriteria로 분리, Schema/DTO 패턴과 일관성 확보, 상속 패턴으로 DRY 유지
- 1.7.1 (2026-01-11): UserQueryCriteria → UserCriteria 단순화, search/filter/paging/order 필드 명확화
- 1.7.0 (2026-01-11): 모듈 네이밍을 Models & Criteria로 정리, ORM Entity 위치를 infra/orm으로 명확화, QueryCondition → QueryCriteria 일관화
- 1.6.0 (2026-01-11): Domain Model 네이밍 규칙 추가 (2.4) - DDD 엄격 적용 시 비즈니스 로직 모델 분리
- 1.5.0 (2026-01-11): Entity 네이밍 규칙 추가 (2.3) - 데이터 계층 모델 패턴 (Create/Update/Delete Model 분리)
- 1.4.0 (2026-01-11): DTO 클래스 네이밍 컨벤션 추가 - CQRS 패턴 (Command/Query) 기반
- 1.3.1 (2026-01-11): ListResponse/PagedListResponse 필드명을 items로 통일 (완전한 일관성 확보)
- 1.3.0 (2026-01-11): 대량 작업(Bulk Operations) 패턴 추가 - URL 액션 기반 네이밍
- 1.2.0 (2026-01-11): GET 패턴에도 Get 접두사 추가 (모든 HTTP 메서드 완전 일관성 확보)
- 1.1.0 (2026-01-11): Schema 클래스 네이밍 컨벤션 추가 (HTTP 메서드별 Request/Response 패턴)
- 1.0.1 (2026-01-11): Router 네이밍을 복수형에서 단수형으로 변경 (전체 레이어 일관성 확보)
- 1.0.0 (2026-01-11): 초기 버전 작성 — 모듈(파일) 네이밍 컨벤션 정의
