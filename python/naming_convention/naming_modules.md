# Python 네이밍 컨벤션 — 모듈(파일)

문서 탐색: [모듈(파일)](naming_modules.md) · [Schema & DTO](naming_schema_dto.md) · [Repository/Entity](naming_repository_model.md) · [Domain Model](naming_domain_model.md) · [요약 & 변경 이력](naming_summary.md)

## 1. 모듈(파일) 네이밍

### 기본 규칙
- **소문자 + 언더스코어** (snake_case) 사용
- 짧고 명확하며 설명적인 이름
- 약어는 가급적 피하고, 사용 시 소문자로 표기 (예: `http_client.py`, `db_config.py`)

### 1.6 Import 경로 규칙
- **패키지 레벨 import**: 최하위 모듈명을 경로에 노출하지 않고 패키지 레벨까지만 import합니다.
  - 권장: `from app.modules.baseinfo.application.dtos import ClassName`
  - 비권장: `from app.modules.baseinfo.application.dtos.baseinfo_category_dto import ClassName`
- **__init__.py 활용**: 각 패키지의 `__init__.py`에서 하위 모듈을 export하여 짧은 경로를 유지합니다.
- **일관성**: 동일 레이어/도메인에서는 동일한 import 깊이를 맞춥니다.

### 레이어별 네이밍 패턴

#### 1.1 Models & Criteria (domain/models/)
- **단수형** 명사 사용, 접미사 없이 리소스 이름만 사용
- **조회 조건은 `_criteria` 접미사**를 사용해 `domain/models/criteria/` 아래에 분리
- ORM Entity는 **infra/orm/** 등 내부 네임스페이스에 숨기고, `domain/models/`에는 도메인/쿼리 모델만 둡니다.

```
✅ 권장
domain/models/user.py
domain/models/order.py
domain/models/criteria/user_criteria.py              # 단순 목록 조회
domain/models/criteria/user_paged_criteria.py        # 페이징 목록 조회
domain/models/criteria/order_criteria.py

❌ 지양
domain/entities/user.py        # domain/entities 대신 domain/models 사용
domain/models/users.py         # 복수형 지양
domain/models/user_model.py    # 불필요한 접미사
domain/models/criteria/user_condition.py # Criteria 접미사 불일치
```

#### 1.2 Protocols (domain/protocols/)
- **단수형 명사 + `_protocol`** 접미사
- Repository/Client 등 인터페이스 정의

```
✅ 권장
domain/protocols/user_repository_protocol.py
domain/protocols/payment_client_protocol.py
domain/protocols/notification_service_protocol.py

❌ 지양
domain/protocols/user_repo.py           # 불명확한 약어
domain/protocols/users_repository.py    # 복수형 지양
```

#### 1.3 Services (application/services/, domain/services/)
- **단수형 명사 + `_app_service` / `_int_service` / `_domain_service` / `_query_service`** 접미사
- 서비스 유형을 명확히 구분

```
✅ 권장 (AppService)
application/services/user_app_service.py
application/services/order_app_service.py

✅ 권장 (IntService)
application/services/payment_int_service.py
application/services/notification_int_service.py

✅ 권장 (QueryService)
application/services/user_query_service.py
application/services/order_query_service.py

✅ 권장 (DomainService)
domain/services/order_pricing_domain_service.py
domain/services/user_validation_domain_service.py

❌ 지양
application/services/user_service.py      # 서비스 유형 불명확
application/services/users_app_service.py # 복수형 지양
application/services/user_query.py        # 접미사 누락
```

#### 1.4 Repositories (infra/repositories/)
- **단수형 명사 + `_repository`** 접미사
- Protocol 구현체임을 명시

```
✅ 권장
infra/repositories/user_repository.py
infra/repositories/order_repository.py
infra/repositories/payment_repository.py

❌ 지양
infra/repositories/user_repo.py    # 불명확한 약어
infra/repositories/users.py        # 복수형 및 접미사 누락
```

#### 1.5 DTOs (application/dtos/)
- **단수형 명사 + `_dto`** 접미사
- 유스케이스/기능별로 그룹화 가능

```
✅ 권장
application/dtos/user_dto.py
application/dtos/order_dto.py
application/dtos/payment_request_dto.py   # 세부 유형 포함 가능

❌ 지양
application/dtos/user.py           # 접미사 누락
application/dtos/users_dto.py      # 복수형 지양
```

#### 1.6 Schemas (interface/schemas/)
- **단수형 명사 + `_schema`** 접미사
- HTTP 요청/응답 검증용

```
✅ 권장
interface/schemas/user_schema.py
interface/schemas/order_schema.py
interface/schemas/auth_schema.py

❌ 지양
interface/schemas/user.py          # 접미사 누락
interface/schemas/users_schema.py  # 복수형 지양
```

#### 1.7 Routers (interface/routers/)
- **단수형 명사 + `_router`** 접미사
- 클라이언트별 분리 시 경로에 반영 (`client/manager/`, `client/user/`)
- REST 엔드포인트가 복수형이어도 파일명은 단수형 유지 (일관성)

```
✅ 권장
interface/routers/user_router.py
interface/routers/order_router.py
interface/routers/client/manager/user_router.py

❌ 지양
interface/routers/user.py          # 접미사 누락
interface/routers/users_router.py  # 복수형 지양
```

#### 1.8 Clients (infra/clients/)
- **단수형 명사 + `_client`** 접미사
- 외부 시스템 연동 클라이언트

```
✅ 권장
infra/clients/payment_client.py
infra/clients/notification_client.py
infra/clients/email_client.py

❌ 지양
infra/clients/payment.py           # 접미사 누락
infra/clients/payments_client.py   # 복수형 지양
```

#### 1.9 Core & Utils

```
✅ 권장
core/config.py
core/dependencies.py
core/security.py
utils/datetime_utils.py
utils/string_utils.py

❌ 지양
core/conf.py               # 불명확한 약어
utils/helpers.py           # 너무 포괄적인 이름
```
