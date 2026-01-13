# Python 데이터 포맷 표준

이 문서는 Python 백엔드 프로젝트에서 데이터를 저장하고 전송할 때 사용하는 포맷 표준을 정의합니다.

## 목차

1. [날짜/시간 포맷](#날짜시간-포맷)
2. [ID 포맷](#id-포맷)
3. [통화 및 숫자](#통화-및-숫자)
4. [구현 예시](#구현-예시)

## 날짜/시간 포맷

### 표준 포맷: ISO 8601 with Timezone

모든 날짜/시간 데이터는 **ISO 8601 포맷 with timezone**을 사용합니다.

#### 기본 원칙

1. **저장 형식**: ISO 8601 문자열 (`YYYY-MM-DDTHH:MM:SS.ffffff+HH:MM` 또는 `Z`)
2. **타임존**: 항상 타임존 정보를 포함 (UTC 권장)
3. **Database**: `TIMESTAMP WITH TIME ZONE` 또는 문자열로 저장
4. **Python Type**: `datetime.datetime` (timezone-aware)
5. **JSON 직렬화**: ISO 8601 문자열로 변환

#### 포맷 예시

```python
# UTC 타임존 (권장)
"2026-01-14T12:30:45.123456+00:00"
"2026-01-14T12:30:45.123456Z"

# 다른 타임존
"2026-01-14T21:30:45.123456+09:00"  # KST (한국 표준시)
"2026-01-14T05:30:45.123456-07:00"  # PDT (태평양 일광 절약 시간)
```

#### Python 구현

```python
from datetime import datetime, timezone

# 1. datetime 객체 생성 (timezone-aware)
now_utc = datetime.now(timezone.utc)

# 2. ISO 8601 문자열로 변환
iso_string = now_utc.isoformat()
# 결과: "2026-01-14T12:30:45.123456+00:00"

# 3. ISO 8601 문자열을 datetime으로 파싱
parsed_dt = datetime.fromisoformat("2026-01-14T12:30:45.123456+00:00")

# 4. 타임존 변환
from zoneinfo import ZoneInfo

# UTC를 KST로 변환
kst_time = now_utc.astimezone(ZoneInfo("Asia/Seoul"))
```

#### Pydantic 스키마

```python
from datetime import datetime
from pydantic import BaseModel, Field

class UserResponse(BaseModel):
    id: int
    username: str
    created_at: datetime  # timezone-aware datetime
    updated_at: datetime | None = None
    
    class Config:
        # JSON 직렬화 시 ISO 8601 문자열로 자동 변환
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }

# 사용 예시
user = UserResponse(
    id=1,
    username="john",
    created_at=datetime.now(timezone.utc)
)

# JSON 직렬화
json_data = user.model_dump_json()
# {"id": 1, "username": "john", "created_at": "2026-01-14T12:30:45.123456+00:00"}
```

#### SQLAlchemy ORM

```python
from datetime import datetime, timezone
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    username = Column(String(100), nullable=False)
    
    # Option 1: TIMESTAMP WITH TIME ZONE (PostgreSQL)
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(timezone.utc)
    )
    
    # Option 2: TIMESTAMP WITHOUT TIME ZONE (다른 DB)
    # 저장 전에 UTC로 정규화하여 저장
    created_at_normalized = Column(
        DateTime(timezone=False),
        nullable=False,
        default=lambda: datetime.now(timezone.utc).replace(tzinfo=None)
    )
```

#### Domain Entity

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone

@dataclass(slots=True)
class User:
    id: int | None = None
    username: str
    email: str
    created_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime | None = None
    
    def to_dict(self) -> dict:
        """Entity를 dict로 변환 (ISO 8601 문자열 포함)"""
        return {
            "id": self.id,
            "username": self.username,
            "email": self.email,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat() if self.updated_at else None,
        }
```

### 날짜/시간 관련 권장사항

#### 1. 항상 UTC를 기본으로 사용

```python
from datetime import datetime, timezone

# ✅ 권장: UTC timezone-aware
utc_now = datetime.now(timezone.utc)

# ❌ 비권장: naive datetime (타임존 정보 없음)
naive_now = datetime.now()

# ❌ 비권장: utcnow() (deprecated in Python 3.12+)
utc_now_old = datetime.utcnow()
```

#### 2. 타임존 변환은 표시 시점에만

- 저장/처리: 항상 UTC
- 표시: 클라이언트에서 사용자 타임존으로 변환

```python
from zoneinfo import ZoneInfo

# 서버: UTC로 저장
utc_time = datetime.now(timezone.utc)

# 클라이언트로 전송 (ISO 8601 문자열)
api_response = {"created_at": utc_time.isoformat()}

# 클라이언트: 사용자 타임존으로 변환 (프론트엔드에서 처리)
# JavaScript: new Date("2026-01-14T12:30:45.123456Z").toLocaleString()
```

#### 3. Database 저장 방식

**PostgreSQL**: `TIMESTAMP WITH TIME ZONE` 권장
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**MySQL/MariaDB**: `TIMESTAMP` (UTC 자동 변환)
```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**SQLite**: 문자열로 저장 (ISO 8601)
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

#### 4. API 응답 형식

OpenAPI 스펙에서는 `format: date-time` (RFC 3339) 사용:

```yaml
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        username:
          type: string
        created_at:
          type: string
          format: date-time  # RFC 3339 = ISO 8601 subset
          example: "2026-01-14T12:30:45.123456Z"
```

## ID 포맷

리소스 식별자는 **ULID** 사용을 권장합니다.

- **장점**: 시간 기반 정렬 가능, URL-safe, 충돌 확률 낮음
- **포맷**: 26자 문자열 (예: `01HQZX7J9K2M3N4P5Q6R7S8T9V`)
- **대안**: UUID도 허용하되 프로젝트 시작 시 명확히 결정

```python
# ULID 사용 예시 (python-ulid 라이브러리)
from ulid import ULID

user_id = ULID()
print(str(user_id))  # "01HQZX7J9K2M3N4P5Q6R7S8T9V"
```

## 통화 및 숫자

### 금액 (Currency)

- **Python**: `Decimal` 타입 사용 (float 사용 금지)
- **Database**: `NUMERIC` 또는 `DECIMAL` 타입
- **API**: 문자열 또는 숫자 (precision 명시)

```python
from decimal import Decimal

price = Decimal("19.99")
```

## 구현 예시

### 전체 예시: User Entity

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from decimal import Decimal

@dataclass(slots=True)
class User:
    """User domain entity with standardized data formats"""
    
    id: int | None = None
    username: str
    email: str
    balance: Decimal = Decimal("0.00")
    
    # 날짜/시간: ISO 8601 with timezone (UTC)
    created_at: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime | None = None
    last_login_at: datetime | None = None
    
    def to_dict(self) -> dict:
        """Serialize to dictionary with ISO 8601 datetime strings"""
        return {
            "id": self.id,
            "username": self.username,
            "email": self.email,
            "balance": str(self.balance),
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat() if self.updated_at else None,
            "last_login_at": self.last_login_at.isoformat() if self.last_login_at else None,
        }
```

### Pydantic 스키마 예시

```python
from datetime import datetime
from decimal import Decimal
from pydantic import BaseModel, Field, ConfigDict

class UserCreateRequest(BaseModel):
    """User creation request schema"""
    username: str = Field(..., min_length=3, max_length=100)
    email: str = Field(..., pattern=r"^[^@]+@[^@]+\.[^@]+$")
    initial_balance: Decimal = Field(default=Decimal("0.00"), ge=0)

class UserResponse(BaseModel):
    """User response schema with ISO 8601 datetime"""
    model_config = ConfigDict(
        json_encoders={
            datetime: lambda v: v.isoformat(),
            Decimal: lambda v: str(v)
        }
    )
    
    id: int
    username: str
    email: str
    balance: Decimal
    created_at: datetime
    updated_at: datetime | None = None
    last_login_at: datetime | None = None
```

## 참고 문서

- [API 스펙 작성 표준](../API_SPEC_STANDARDS.md) - RFC 3339/ISO 8601 포맷
- [Database 표준](DATABASE_STANDARDS.md) - TIMESTAMP WITH TIME ZONE 사용
- ISO 8601: https://en.wikipedia.org/wiki/ISO_8601
- RFC 3339: https://tools.ietf.org/html/rfc3339
- Python datetime: https://docs.python.org/3/library/datetime.html

## 버전 이력

- 1.0.0 (2026-01-14): 초기 버전 작성 - ISO 8601 with timezone 표준 정의
