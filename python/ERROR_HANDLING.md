# Python 에러 처리 표준 (FastAPI / DDD)

목표: 일관된 예외 계층, 응답 포맷(code/message/data), 로깅/트레이싱 규칙을 정의하여 재현성과 디버깅 가능성을 높인다.

## 1. 원칙
- 예외는 **의도된 계층 전파**만 허용한다: Domain → Application → Interface. Infra 예외는 상위 계층 전파 전 래핑한다.
- HTTP 경계에서는 `HTTPException` 대신 **표준 응답(code/message/data)**로 변환한다.
- 로깅은 **한 번만**: Interface 레이어에서 최종적으로 기록하고 중간 레이어는 로깅하지 않는다(중복 방지).
- 재시도는 **멱등 작업**에만, 짧고 제한된 횟수로 수행한다.

## 2. 예외 계층
```
BaseAppError(Exception)
├── DomainError              # 비즈니스 규칙 위반 (도메인 계층 전용)
├── ApplicationError         # 애플리케이션/유스케이스 로직 실패
├── InfraError               # DB/외부 API/메시지 브로커 등 인프라 장애
└── ValidationError          # 스키마/DTO/입력 검증 실패 (Pydantic)
```

### 2.1 DomainError
- 도메인 순수 규칙 위반에만 사용. HTTP/DB 세부사항 금지.
- 예시: `InvalidEmailError`, `InsufficientBalanceError`.

### 2.2 ApplicationError
- 트랜잭션 경계 안에서 발생한 유스케이스/애플리케이션 로직 실패를 표현. 재시도하지 않는 논리 오류.
- 예시: `DuplicateUserError`, `UnauthorizedActionError`.

### 2.3 InfraError
- 드라이버/네트워크/쿼리 타임아웃 등 기술적 실패를 래핑.
- 예시: `DbTimeoutError`, `ExternalServiceError`.

### 2.4 ValidationError
- Pydantic/입력 검증 실패. Interface에서 400으로 변환.

## 3. 계층별 사용 규칙
- **Domain**: DomainError만 발생시킨다. HTTP/DB 지식 없음.
- **Application**: Domain/Infra 예외를 잡아 ApplicationError로 전환하거나 그대로 전파. HTTPException 사용 금지.
- **Infra**: 드라이버 예외를 InfraError로 래핑 후 전파. 로깅하지 않는다.
- **Interface (FastAPI)**: 모든 예외를 표준 응답으로 변환. 여기서 단일 로깅 수행.
- **메시지 마스킹**: InfraError/미처리 예외(5xx)는 사용자 메시지를 "Internal server error" 등 일반 문구로 제한하고, 상세 원인은 서버 로그에만 남긴다.

## 4. 표준 응답 포맷 (재확인)
```json
{
  "code": "USER_DUPLICATED",
  "message": "User already exists",
  "data": null
}
```
- 성공 시 `code="OK"`, `message="success"`, `data`에 페이로드.
- 실패 시 `code`는 도메인/유스케이스/인프라 식별 가능한 문자열, `message`는 사용자 전달용.
- 에러 코드 명명 규칙: `DOMAIN_ACTION_REASON` (예: `USER_CREATE_DUPLICATED`, `ORDER_CANCEL_EXPIRED`). 코드만으로 도메인/행위/사유를 식별한다.

## 5. FastAPI 전역 핸들러 예시
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from shared.errors import DomainError, ApplicationError, InfraError
from pydantic import ValidationError as PydanticValidationError

app = FastAPI()

@app.exception_handler(DomainError)
async def handle_domain_error(request: Request, exc: DomainError):
    return JSONResponse(status_code=400, content={"code": exc.code, "message": str(exc), "data": None})

@app.exception_handler(ApplicationError)
async def handle_application_error(request: Request, exc: ApplicationError):
    return JSONResponse(status_code=400, content={"code": exc.code, "message": str(exc), "data": None})

@app.exception_handler(InfraError)
async def handle_infra_error(request: Request, exc: InfraError):
    # 5xx: 클라이언트 노이즈를 줄이기 위해 일반 메시지 사용
    return JSONResponse(status_code=503, content={"code": exc.code, "message": "Temporary service error", "data": None})

@app.exception_handler(PydanticValidationError)
async def handle_validation_error(request: Request, exc: PydanticValidationError):
    # Pydantic errors()는 중첩 구조이므로 요약/필드 기준으로 가공해 전달
    simplified = [
        {
            "loc": ".".join(map(str, err.get("loc", []))),
            "msg": err.get("msg", "validation error"),
            "type": err.get("type", "")
        }
        for err in exc.errors()
    ]
    return JSONResponse(status_code=400, content={"code": "INVALID_REQUEST", "message": simplified, "data": None})
```

## 6. 로깅 & 트레이싱
- 로깅 위치: Interface 전역 핸들러에서만. Domain/Application/Infra에서는 로깅 금지.
- 필수 필드: `trace_id`/`request_id`, `exception_type`, `code`, `path`, `method`, `elapsed_ms`.
- 4xx(클라이언트 오류)는 `INFO` 또는 `WARNING`, 5xx는 `ERROR` 레벨.

## 7. 재시도 정책
- 조건: 멱등 요청(예: GET, idempotent PUT) + 명확한 재시도 가능 오류(타임아웃, 일시적 네트워크 실패).
- 제한: 최대 3회, 지수 백오프(예: 0.1s, 0.3s, 0.9s), 전체 타임아웃 상한.
- 금지: 트랜잭션 내부에서 동일 세션으로 재시도, 비멱등 POST/DELETE, 외부 API에서 중복 부작용 유발 시.

## 8. 트랜잭션과 에러
- `async with transaction:` 블록 내에서 예외 발생 시 자동 롤백(트랜잭션 패턴 문서 준수).
- Application은 `InfraError`를 필요 시 `ApplicationError`로 변환 후 전파, Interface에서 5xx로 매핑.

## 9. HTTP 매핑 가이드라인
| 예외 | HTTP | code 예시 | message 예시 |
|------|------|-----------|--------------|
| ValidationError | 400 | INVALID_REQUEST | Validation detail 배열 or 요약 메시지 |
| DomainError | 400 | DOMAIN_RULE_VIOLATION | 비즈니스 규칙 위반 메시지 |
| ApplicationError | 400 | APPLICATION_FAILED | 유스케이스/애플리케이션 실패 메시지 |
| InfraError | 503 | SERVICE_UNAVAILABLE | 일반화된 사용자 메시지 |
| 미처리 예외 | 500 | UNEXPECTED_ERROR | "Internal server error" |

## 10. 커스텀 예외 예시
```python
class BaseAppError(Exception):
    code: str = "APP_ERROR"
    def __init__(self, message: str, origin_exc: Exception | None = None):
        super().__init__(message)
        self.origin_exc = origin_exc  # 디버깅/로깅용 (응답에는 노출하지 않음)

class InvalidEmailError(BaseAppError):
    code = "INVALID_EMAIL"

class DuplicateUserError(BaseAppError):
    code = "USER_DUPLICATED"

class DbTimeoutError(BaseAppError):
    code = "DB_TIMEOUT"
```

## 11. 테스트 가이드
- 단위 테스트: `pytest.raises`로 예외 타입과 `code`를 검증.
- API 테스트: 응답 JSON에서 `code/message/data` 확인, HTTP 상태와 코드 매핑을 검증.
- 재시도 로직: 멱등 조건을 충족하는지, 백오프가 적용되는지 테스트.

## 12. 체크리스트
- [ ] Domain은 DomainError만, Infra는 InfraError만 발생시키는가?
- [ ] Application에서 HTTPException을 사용하지 않는가?
- [ ] 전역 핸들러가 code/message/data 포맷을 반환하는가?
- [ ] 로깅이 Interface 한 곳에서만 수행되는가?
- [ ] 재시도 범위가 멱등 작업으로 한정되는가?
