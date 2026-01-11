# Python 네이밍 컨벤션 — Schema & DTO

문서 탐색: [모듈(파일)](naming_modules.md) · [Schema & DTO](naming_schema_dto.md) · [Repository/Entity](naming_repository_model.md) · [Domain Model](naming_domain_model.md) · [요약 & 변경 이력](naming_summary.md)

## 2. 클래스 네이밍

### 2.1 Schema 클래스 (interface/schemas/)

Schema는 HTTP 요청/응답 검증을 위한 Pydantic 모델입니다. REST API 메서드와 리소스 타입에 따라 명확한 네이밍 패턴을 사용합니다.

#### 기본 원칙
- **PascalCase** 사용
- **Resource는 단수형** (User, Order)
- **HTTP 메서드 접두사 명시** (Post, Put, Delete 등)
- **Request/Response 접미사 필수**

#### HTTP 메서드별 패턴

##### POST (생성)
- 패턴: `Post{Resource}Request`, `Post{Resource}Response`
- 예시:
```python
class PostUserRequest(BaseModel):
    email: str
    name: str

class PostUserResponse(BaseModel):
    id: int
    email: str
    name: str
    created_at: datetime
```

##### PUT (전체 수정)
- 패턴: `Put{Resource}Request`, `Put{Resource}Response`

##### PATCH (부분 수정)
- 패턴: `Patch{Resource}Request`, `Patch{Resource}Response`

##### DELETE (삭제)
- 패턴: `Delete{Resource}Request`, `Delete{Resource}Response`
- Request는 path/query parameter일 때 생략 가능

##### GET (단건 조회)
- 패턴: `Get{Resource}Request`, `Get{Resource}Response`
- Request는 query parameter가 있는 경우만 사용

##### GET (목록 조회 - 단순)
```python
class GetUserListRequest(BaseModel):
    status: str | None = None

class GetUserListItemInfo(BaseModel):
    id: int
    email: str
    name: str

class GetUserListResponse(BaseModel):
    items: list[GetUserListItemInfo]
```

##### GET (목록 조회 - 페이징)
- `items: list[Get{Resource}PagedListItemInfo]`
- `total_count: int`
```python
class GetUserPagedListRequest(BaseModel):
    page: int = 1
    page_size: int = 20
    status: str | None = None
```

#### 대량 작업 (Bulk Operations)
- 패턴: `{HttpMethod}{Resource}{작업수식어}Request/Response`
- URL 액션명을 그대로 사용 (`:bulk` → `Bulk`, `:upsert` → `Upsert`)

#### 하위 타입 (Nested Objects)
- 패턴: `{Verb}{Resource}Request{SubType}Info`
- Request/Response 본문 내부의 중첩 객체/컬렉션 아이템

#### Schema 요약 테이블

| HTTP 메서드 | Request 패턴 | Response 패턴 | 예시 |
|-------------|--------------|---------------|------|
| POST | `Post{Resource}Request` | `Post{Resource}Response` | `PostUserRequest` |
| PUT | `Put{Resource}Request` | `Put{Resource}Response` | `PutUserRequest` |
| PATCH | `Patch{Resource}Request` | `Patch{Resource}Response` | `PatchUserRequest` |
| DELETE | `Delete{Resource}Request` | `Delete{Resource}Response` | `DeleteUserResponse` |
| GET (단건) | `Get{Resource}Request` | `Get{Resource}Response` | `GetUserResponse` |
| GET (목록) | `Get{Resource}ListRequest` | `Get{Resource}ListResponse` | `GetUserListResponse` |
| GET (페이징) | `Get{Resource}PagedListRequest` | `Get{Resource}PagedListResponse` | `GetUserPagedListResponse` |
| 대량 작업 | `{HttpMethod}{Resource}{작업수식어}Request` | `{HttpMethod}{Resource}{작업수식어}Response` | `PostUserBulkRequest` |
| 하위 타입 | `{Verb}{Resource}Request{SubType}Info` | - | `PostUserRequestPermissionInfo` |

---

### 2.1.1 응답 구조 표준 (Response Body Format)

모든 API 응답은 일관된 래핑 구조를 사용합니다. 성공/실패 모두 동일한 필드 구조로 클라이언트 파싱 로직을 단순화합니다.

#### 기본 원칙
- **모든 응답**은 `code`, `message`, `data` 필드를 가짐
- **HTTP Status**로 성공/실패 1차 구분
- **Body `code`**로 비즈니스 세부 상태 표현 (문자열, UPPER_SNAKE_CASE)
- 성공/실패 모두 `data`로 통일 (일관성)

#### 응답 구조

```python
class ApiResponse(BaseModel, Generic[T]):
    """공통 API 응답 래퍼"""
    code: str          # 비즈니스 코드 (SUCCESS, CREATED, USER_EMAIL_DUPLICATED 등)
    message: str       # 사람이 읽을 수 있는 메시지
    data: T | None     # 성공 시 리소스/결과, 실패 시 에러 상세 정보
```

#### 성공 응답 예시

##### POST (생성, 201)
```python
# HTTP 201 Created
{
  "code": "CREATED",
  "message": "User created successfully",
  "data": {
    "id": 123,
    "email": "user@example.com",
    "name": "John Doe",
    "created_at": "2026-01-11T10:30:00Z"
  }
}
```

##### GET (조회, 200)
```python
# HTTP 200 OK
{
  "code": "SUCCESS",
  "message": "User retrieved",
  "data": {
    "id": 123,
    "email": "user@example.com",
    "name": "John Doe"
  }
}
```

##### GET (목록/페이징, 200)
```python
# HTTP 200 OK
{
  "code": "SUCCESS",
  "message": "User list retrieved",
  "data": {
    "items": [
      {"id": 1, "email": "user1@example.com"},
      {"id": 2, "email": "user2@example.com"}
    ],
    "total_count": 100,
    "page": 1,
    "page_size": 20
  }
}
```

##### PUT/PATCH (수정, 200)
```python
# HTTP 200 OK
{
  "code": "UPDATED",
  "message": "User updated successfully",
  "data": {
    "id": 123,
    "email": "newemail@example.com",
    "name": "John Doe",
    "updated_at": "2026-01-11T11:00:00Z"
  }
}
```

##### DELETE (삭제, 200 또는 204)
```python
# HTTP 200 OK (본문 포함)
{
  "code": "DELETED",
  "message": "User deleted successfully",
  "data": {
    "id": 123,
    "deleted_at": "2026-01-11T11:30:00Z"
  }
}

# HTTP 204 No Content (본문 없음) - 선택 가능
```

#### 에러 응답 예시

##### 400 Bad Request (유효성 검증 실패)
```python
# HTTP 400 Bad Request
{
  "code": "VALIDATION_ERROR",
  "message": "Invalid input data",
  "data": {
    "errors": [
      {"field": "email", "message": "Invalid email format"},
      {"field": "name", "message": "Name is required"}
    ]
  }
}
```

##### 400 Bad Request (비즈니스 규칙 위반)
```python
# HTTP 400 Bad Request
{
  "code": "USER_EMAIL_DUPLICATED",
  "message": "Email already exists",
  "data": {
    "field": "email",
    "value": "user@example.com"
  }
}
```

##### 401 Unauthorized
```python
# HTTP 401 Unauthorized
{
  "code": "AUTHENTICATION_FAILED",
  "message": "Invalid credentials",
  "data": null
}
```

##### 403 Forbidden
```python
# HTTP 403 Forbidden
{
  "code": "PERMISSION_DENIED",
  "message": "You do not have permission to access this resource",
  "data": {
    "required_permission": "admin",
    "user_role": "user"
  }
}
```

##### 404 Not Found
```python
# HTTP 404 Not Found
{
  "code": "RESOURCE_NOT_FOUND",
  "message": "User not found",
  "data": {
    "resource_type": "User",
    "resource_id": 999
  }
}
```

##### 409 Conflict
```python
# HTTP 409 Conflict
{
  "code": "RESOURCE_CONFLICT",
  "message": "Optimistic lock failure",
  "data": {
    "expected_version": 5,
    "actual_version": 6
  }
}
```

##### 500 Internal Server Error
```python
# HTTP 500 Internal Server Error
{
  "code": "INTERNAL_SERVER_ERROR",
  "message": "An unexpected error occurred",
  "data": {
    "trace_id": "abc123def456"
  }
}
```

#### 비즈니스 코드 (code) 네이밍 규칙
- **형식**: UPPER_SNAKE_CASE 문자열
- **도메인 접두사 적용 대상**: 비즈니스 에러/비정상 상태 코드에 `{DOMAIN}_{상태}` 패턴 사용  
  - 예: `USER_EMAIL_DUPLICATED`, `ORDER_ALREADY_SHIPPED`
- **성공 코드**: 전역 공통 코드로 도메인 접두사를 붙이지 않음  
  - 예: `SUCCESS`, `CREATED`, `UPDATED`, `DELETED`, `PARTIAL_SUCCESS`
- **에러 코드**: 도메인 접두사를 포함한 구체적이고 의미 있는 식별자 (HTTP Status와 중복 X)
  - 예: `USER_EMAIL_DUPLICATED`, `ORDER_ALREADY_SHIPPED`, `PAYMENT_METHOD_EXPIRED`

#### 클라이언트 처리 규칙

1. **1단계: HTTP Status 확인**
   - `2xx`: 성공 → `data`에서 리소스 파싱
   - `4xx/5xx`: 에러 → `code`로 세부 에러 분기, `data`에서 에러 상세 정보 파싱

2. **2단계: Body `code` 활용**
   - 성공 시: `code`로 추가 액션 판단 (`PARTIAL_SUCCESS`, `CREATED` 등)
   - 에러 시: `code`로 UX 분기 (메시지 커스터마이징, 재시도 로직 등)

3. **3단계: `data` 파싱**
   - 성공: 리소스 객체/목록
   - 에러: 에러 상세 정보 (필드, 값, 추가 컨텍스트)

#### 클라이언트 예시 (TypeScript)

```typescript
if (response.status >= 400) {
  // 에러 처리
  switch (response.data.code) {
    case 'USER_EMAIL_DUPLICATED':
      showError('이미 사용 중인 이메일입니다.');
      break;
    case 'PERMISSION_DENIED':
      redirectToLogin();
      break;
    default:
      showError(response.data.message);
  }
} else {
  // 성공 처리
  const user = response.data.data;
  console.log('User created:', user.id);
}
```

---

### 2.2 DTO 클래스 (application/dtos/)

DTO는 Application Layer와 다른 레이어 간 데이터 전송을 위한 모델입니다. CQRS 패턴을 따라 Command(CUD)와 Query(R)로 명확히 구분합니다.

#### 기본 원칙
- **PascalCase** 사용
- **Resource는 단수형** (User, Order)
- **CQRS 패턴**: Command(상태 변경), Query(상태 조회) 분리
- **Schema와의 구분**: Command/Query vs Request/Response로 레이어 책임 명확화

#### CUD 작업 (Command 패턴)
- 등록: `Register{Resource}Command` / `Register{Resource}CommandResult`
- 수정: `Update{Resource}Command` / `Update{Resource}CommandResult`
- 삭제: `Delete{Resource}Command` / `Delete{Resource}CommandResult`

#### R 작업 (Query 패턴)
- 단건 조회: `{resource}_id` (직접 전달), 응답은 `{Resource}Dto`
- 목록 조회: `{Resource}ListQuery` / `{Resource}ListQueryResult`
- 페이징 조회: `{Resource}PagedListQuery` / `{Resource}PagedListQueryResult`

#### 하위 타입 (Nested Objects)
- 패턴: `{Command/Query}{Resource}{SubType}Info`

#### DTO 요약 테이블

| 작업 유형 | 입력 패턴 | 출력 패턴 | 예시 |
|----------|----------|----------|------|
| 등록 | `Register{Resource}Command` | `Register{Resource}CommandResult` | `RegisterUserCommand` |
| 수정 | `Update{Resource}Command` | `Update{Resource}CommandResult` | `UpdateUserCommand` |
| 삭제 | `Delete{Resource}Command` | `Delete{Resource}CommandResult` | `DeleteUserCommand` |
| 단건 조회 | `{resource}_id: int` | `{Resource}Dto` | `user_id → UserDto` |
| 목록 조회 | `{Resource}ListQuery` | `{Resource}ListQueryResult` | `UserListQuery` |
| 페이징 조회 | `{Resource}PagedListQuery` | `{Resource}PagedListQueryResult` | `UserPagedListQuery` |
| 하위 타입 | `{Command/Query}{SubType}Info` | - | `RegisterUserCommandPermissionInfo` |

---

### 2.3 Entity ↔ DTO 분리의 장점 (가이드)

Schema/DTO와 Entity를 분리하면 다음과 같은 이점이 있습니다. (세부 예시는 프로젝트별 문서/코드 참조)

- **계층 간 독립적 진화**: DB 스키마(Entity) 변경 시에도 API 응답(DTO)은 유지 가능.
- **데이터 노출 제어**: 민감 데이터는 Entity에만 존재, DTO에는 노출 금지로 안전한 응답.
- **정규화/비정규화 유연성**: 성능을 위해 비정규화(엔티티)하더라도, DTO는 정규 구조 유지.
- **ORM 활용 이점**: 관계/캐싱 등 ORM 고급 기능은 인프라에서 활용, DTO는 단순 유지.
- **테스트 용이성**: DTO 중심으로 비즈니스 테스트 작성 → Entity 변경 영향 최소화.
- **마이그레이션 안전성**: 스키마 버전 변경에도 DTO/Schema는 일관성 유지.

권장 흐름:
1) Router에서 Schema 검증 → DTO 변환
2) AppService에서 비즈니스 처리 → 필요 시 Domain Model 사용
3) Repository 구현에서 Entity ↔ DTO/Domain 변환을 캡슐화
