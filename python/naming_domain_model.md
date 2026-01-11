# Python 네이밍 컨벤션 — Domain Model

## 2.4 Domain Model 네이밍 규칙 (Optional: DDD 엄격 적용)

**상황**: Domain-Driven Design을 엄격하게 따르거나, 복잡한 비즈니스 로직을 표현해야 할 때, Domain Model을 별도로 정의할 수 있습니다. Domain Model은 **비즈니스 규칙을 캡슐화**하고 DB 영속성과는 완전히 독립적입니다. 도메인 레이어는 Entity(ORM)나 인프라 모델을 알지 못합니다.

### 개념 구분

| 개념 | 위치 | 책임 | 예시 |
|------|------|------|------|
| **Domain Model** | domain/models/ | 비즈니스 규칙, 비즈니스 제약 | `User`는 이메일 유효성, 암호 정책 검증 |
| **DTO** | application/dtos/ | 계층 간 데이터 전달 | `RegisterUserCommand` |
| **Entity** | infra/models/ | DB 스키마 매핑 | SQLAlchemy ORM 모델 |
| **Schema** | interface/schemas/ | HTTP 요청/응답 검증 | `PostUserRequest` |

### 네이밍 규칙
- **PascalCase** 사용
- **Resource는 단수형** (User, Order, Payment)
- **비즈니스 로직을 메서드로 표현** (명사가 아닌 동사 중심)
- **불변성(Immutability) 고려** - 상태 변경은 새로운 인스턴스 반환

#### 생성 (Factory 패턴)
- 정적 메서드로 도메인 모델 생성: `User.create()`, `User.register()`

#### 상태 변경 (Command 메서드)
- 예: `user.change_email()`, `user.update_name()`, `user.deactivate()`

#### 조회 (Query 메서드)
- 예: `user.get_email()`, `user.has_permission()`, `user.is_valid()`

### 데이터 흐름 (Domain Model + Entity)
```
HTTP 요청
→ Schema 검증 (interface)
→ DTO 변환 (application)
→ Domain Model 생성/행동 (domain)
→ Repository 호출 (domain/application 포트)
	↳ Repository 구현(인프라) 내부에서 Domain Model ↔ Entity(ORM) 매핑 및 DB 작업
→ 결과를 Domain Model/DTO로 반환
→ Schema 응답 변환
```

### 계층 구조 비교
```
단순:   Schema → DTO → Repository → DB
중간:   Schema → DTO → Repository → Entity → DB
복잡:   Schema → DTO → Domain Model → Repository → DB
```

### Domain Model vs Entity 분리의 장점
- **비즈니스 규칙 캡슐화**: Domain Model에 로직 집중, Entity는 데이터 보관
- **DB 변경 영향 최소화**: 스키마 변경 시 Domain Model/로직 그대로 유지 가능
- **테스트 독립성**: Domain 로직은 순수 객체로 빠르게 테스트, Entity는 별도
- **역할 명확성**: Domain은 규칙/행동, Entity는 영속성

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
