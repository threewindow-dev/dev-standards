# Python 네이밍 컨벤션 — Schema & DTO

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
