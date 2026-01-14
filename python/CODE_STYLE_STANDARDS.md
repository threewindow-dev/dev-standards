# Python 코드 스타일 표준

문서 탐색: [FastAPI 개발](FASTAPI_DEVELOPMENT_STANDARDS.md) · [네이밍 컨벤션](NAMING_CONVENTION.md) · [에러 처리](ERROR_HANDLING.md)

## 개요

이 문서는 PEP8을 기본으로 하되, 프로젝트에서 추가로 준수해야 할 코드 스타일 규칙을 정의합니다.

**기본 원칙:**
- PEP8을 기본 스타일 가이드로 따릅니다
- PEP8에서 다루지 않거나 명확하지 않은 영역에 대한 추가 규칙을 정의합니다
- 일관성과 가독성을 최우선으로 합니다

## 1. Import 규칙

### 1.1 기본 원칙 (PEP8)

**모든 import는 파일 상단에 위치합니다:**
- 위치: 모듈 docstring 다음, 다른 코드 이전
- 순서: 표준 라이브러리 → 서드파티 → 로컬 모듈
- 그룹 간 빈 줄로 구분

**✅ 권장 예시:**
```python
"""
User domain model.
"""

# 표준 라이브러리
import uuid
from datetime import datetime, timezone
from typing import Optional

# 서드파티
from pydantic import BaseModel

# 로컬 모듈
from shared.errors import DomainError
from domain.protocols import UserRepository
```

### 1.2 함수/메서드 내 Import 금지

**원칙: 모든 import는 파일 상단에 위치하며, 함수/메서드/클래스 내부에서 import하지 않습니다.**

#### ❌ 금지 예시

```python
def create_user(username: str):
    import uuid  # ❌ 불필요한 함수 내 import
    user_id = str(uuid.uuid4())
    return {"id": user_id, "username": username}

class UserService:
    def generate_token(self):
        import secrets  # ❌ 불필요한 메서드 내 import
        return secrets.token_hex(16)
```

#### ✅ 권장 예시

```python
import uuid
import secrets

def create_user(username: str):
    user_id = str(uuid.uuid4())
    return {"id": user_id, "username": username}

class UserService:
    def generate_token(self):
        return secrets.token_hex(16)
```

#### 금지 이유

1. **가독성**: 파일의 모든 의존성을 상단에서 한눈에 파악
2. **성능**: 함수 호출마다 import 체크 오버헤드 방지 (Python은 캐싱하지만 불필요한 검증 발생)
3. **테스트**: mock/patch 시 복잡도 감소
4. **일관성**: 프로젝트 전체 코딩 스타일 통일
5. **도구 지원**: IDE의 자동 import 정리, 미사용 import 탐지 등 활용

### 1.3 예외 케이스 (조건부 허용)

다음의 경우에만 함수/메서드 내 import가 허용됩니다. **반드시 명확한 주석과 함께 사용해야 합니다.**

#### ✅ 1) Circular Import 해결

순환 참조를 피하기 위해 지연 import가 필요한 경우:

```python
# domain/models/user.py
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from domain.models.order import Order

class User:
    def get_orders(self) -> list["Order"]:
        # Circular import 방지를 위한 지연 import
        from domain.models.order import Order
        return Order.find_by_user_id(self.id)
```

#### ✅ 2) 조건부 Import (환경별 모듈)

실행 환경에 따라 다른 모듈을 로드해야 하는 경우:

```python
def get_database_config():
    """환경에 따른 DB 설정 로드"""
    env = os.getenv("ENV", "development")
    
    if env == "production":
        # Production 환경 전용 설정
        from config.production import DatabaseConfig
    elif env == "staging":
        from config.staging import DatabaseConfig
    else:
        from config.development import DatabaseConfig
    
    return DatabaseConfig()
```

#### ✅ 3) Heavy Module Lazy Loading

무거운 라이브러리를 필요할 때만 로드하는 경우:

```python
def process_image(image_path: str):
    """이미지 처리 (OpenCV는 사용 시에만 로드)"""
    # OpenCV는 무거운 라이브러리 - 이 함수 호출 시에만 로드
    import cv2
    import numpy as np
    
    image = cv2.imread(image_path)
    return cv2.resize(image, (640, 480))

def generate_pdf_report(data: dict):
    """PDF 리포트 생성 (reportlab은 사용 시에만 로드)"""
    # reportlab은 크기가 큰 라이브러리
    from reportlab.pdfgen import canvas
    from reportlab.lib.pagesizes import letter
    
    # PDF 생성 로직
    # ...
```

#### ✅ 4) 선택적 의존성 (Optional Dependencies)

특정 기능이 선택적이고, 해당 패키지가 없어도 프로그램이 동작해야 하는 경우:

```python
def export_to_excel(data: list[dict], output_path: str):
    """Excel 파일로 내보내기 (openpyxl이 있는 경우에만)"""
    try:
        import openpyxl
    except ImportError:
        raise RuntimeError(
            "Excel export requires 'openpyxl' package. "
            "Install with: pip install openpyxl"
        )
    
    # Excel 생성 로직
    # ...
```

### 1.4 주석 작성 규칙

예외 케이스에서 함수 내 import를 사용할 때는 **반드시 이유를 주석으로 명시**합니다:

```python
def process_data():
    # Circular import 방지를 위한 지연 import
    from .related_module import RelatedClass
    
    # 또는
    
    # 무거운 ML 라이브러리 - 필요 시에만 로드
    import tensorflow as tf
```

### 1.5 린터 설정

프로젝트에서 사용하는 린터 도구의 권장 설정:

#### Ruff 설정 (pyproject.toml)

```toml
[tool.ruff]
# Import 순서 검증 활성화
select = ["I"]

[tool.ruff.lint]
# 불필요한 import 탐지
ignore = []

[tool.ruff.lint.isort]
known-first-party = ["shared", "core", "subdomains"]
```

#### Flake8 설정 (.flake8)

```ini
[flake8]
max-line-length = 99
exclude = __pycache__,.git,.venv
per-file-ignores =
    # __init__.py 파일에서는 미사용 import 허용 (re-export용)
    __init__.py:F401
```

## 2. 코드 포맷팅

### 2.1 자동 포맷터 사용

**Black을 표준 포맷터로 사용합니다:**

```bash
# 설치
pip install black

# 실행
black backend/src

# CI/CD에서 검증
black --check backend/src
```

**설정 (pyproject.toml):**
```toml
[tool.black]
line-length = 99
target-version = ['py313']
include = '\.pyi?$'
extend-exclude = '''
/(
  # 기본 제외
  \.git
  | __pycache__
  | \.venv
)/
'''
```

### 2.2 줄 길이

- **최대 줄 길이: 99자** (Black 기본값)
- 긴 문자열이나 URL은 예외 허용
- 가독성을 위해 의미 있는 단위로 줄바꿈

## 3. 타입 힌트

### 3.1 기본 원칙

**Python 3.13 문법을 사용합니다:**

```python
# ✅ Python 3.13+ 방식
def get_users(limit: int = 10) -> list[dict[str, str | int]]:
    return [{"id": 1, "name": "John"}]

def find_user(user_id: int) -> User | None:
    return User.find_by_id(user_id)

# ❌ 구버전 방식 (사용 금지)
from typing import List, Dict, Optional, Union

def get_users(limit: int = 10) -> List[Dict[str, Union[str, int]]]:
    return [{"id": 1, "name": "John"}]

def find_user(user_id: int) -> Optional[User]:
    return User.find_by_id(user_id)
```

### 3.2 함수 시그니처

**모든 공개(public) 함수/메서드는 타입 힌트를 작성합니다:**

```python
# ✅ 권장
async def create_user(
    username: str,
    email: str,
    full_name: str | None = None
) -> User:
    """사용자 생성"""
    # ...

# ❌ 비권장
async def create_user(username, email, full_name=None):
    """사용자 생성"""
    # ...
```

### 3.3 복잡한 타입

복잡한 타입은 TypeAlias로 정의합니다:

```python
from typing import TypeAlias

# 타입 별칭 정의
UserId: TypeAlias = int
UserDict: TypeAlias = dict[str, str | int | None]
QueryResult: TypeAlias = tuple[list[User], int]

# 사용
def find_users(skip: int, limit: int) -> QueryResult:
    users = [...]
    total = 100
    return users, total
```

## 4. Docstring

### 4.1 기본 형식

Google 스타일 docstring을 사용합니다:

```python
def create_user(username: str, email: str) -> User:
    """새로운 사용자를 생성합니다.

    Args:
        username: 사용자명 (3-50자)
        email: 이메일 주소

    Returns:
        생성된 User 객체

    Raises:
        DuplicateUserError: 사용자명 또는 이메일이 중복된 경우
        ValidationError: 입력값이 유효하지 않은 경우

    Example:
        >>> user = create_user("john_doe", "john@example.com")
        >>> print(user.username)
        'john_doe'
    """
    # ...
```

### 4.2 작성 규칙

- **모듈**: 파일 상단에 모듈 목적 설명
- **클래스**: 클래스의 역할과 사용법
- **공개 함수/메서드**: Args, Returns, Raises 포함
- **비공개 함수/메서드**: 간단한 설명만 (선택)

## 5. 변수 명명

### 5.1 기본 규칙

```python
# ✅ 권장
user_count = 10
MAX_RETRY_COUNT = 3
_internal_cache = {}

class UserService:
    def __init__(self):
        self._repository = UserRepository()  # protected
        self.__secret_key = "..."  # private
```

### 5.2 Bool 변수

Boolean 변수는 `is_`, `has_`, `can_`, `should_` 등의 접두사 사용:

```python
# ✅ 권장
is_active = True
has_permission = False
can_delete = user.is_admin
should_retry = error_count < MAX_RETRY_COUNT

# ❌ 비권장
active = True  # bool인지 불명확
permission = False
delete = user.is_admin
```

## 6. 코드 리뷰 체크리스트

코드 리뷰 시 다음 항목을 확인합니다:

- [ ] 모든 import가 파일 상단에 위치하는가?
- [ ] 함수 내 import가 있다면 명확한 이유와 주석이 있는가?
- [ ] Python 3.13+ 타입 힌트를 사용하는가? (`list` not `List`, `|` not `Union`)
- [ ] 공개 함수에 docstring이 있는가?
- [ ] Black으로 포맷팅되었는가?
- [ ] 변수명이 명확하고 일관성 있는가?
- [ ] 줄 길이가 99자를 초과하지 않는가?

## 7. 도구 설정 예시

### pyproject.toml (종합)

```toml
[tool.black]
line-length = 99
target-version = ['py313']

[tool.ruff]
line-length = 99
target-version = "py313"
select = ["E", "F", "I", "W"]

[tool.ruff.lint.isort]
known-first-party = ["shared", "core", "subdomains"]

[tool.mypy]
python_version = "3.13"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

## 8. 참고 자료

- [PEP 8 – Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Black Code Style](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html)
- [Google Python Style Guide - Docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)
- [Type hints cheat sheet (Python 3)](https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html)
