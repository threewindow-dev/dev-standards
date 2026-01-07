# API 스펙 작성 표준

목적: API 스펙을 일관성 있게 작성·검증·배포하기 위한 규칙과 체크리스트를 제공합니다. 스펙은 팀의 계약(SSOT)이 될 수 있으므로 변경·검증 절차를 명확히 해야 합니다.

1) 기본 정책
- 스펙 표준 버전: OpenAPI 3.1.0 권장. JSON Schema 2020-12 특성을 사용합니다.
- 스펙 저장 위치: `openapi/` 디렉터리 (예: `openapi/openapi.yaml` 또는 `openapi/<service>-v1.yaml`).
- 파일 형식: YAML 권장(가독성). 다중 서비스인 경우 서비스별 파일로 분리.

2) 파일/버전 네이밍
- 파일명: `<service>-openapi.yaml` 또는 `openapi.yaml` (서비스 루트 레포의 경우)
- 버전 표기: 스펙 `info.version`는 API의 공개 버전(semver 권장).
- URL 버전 정책: 서버 URL에 `/v{major}` 형태 권장(예: `https://api.example.com/v1`).

3) 공통 컴포넌트 규칙
- `components/schemas`에 재사용 가능한 모델을 정의하고 가능한 한 참조(`$ref`) 사용.
- 공통 응답 및 에러: `components/responses`와 `components/schemas/Error`를 정의하여 일관된 에러 포맷을 사용.
- 보안 스키마: `components/securitySchemes`에 표준화(ex: `bearerAuth` JWT, `apiKey` in header)하여 재사용.

4) 응답/에러 형식(권장)
- 성공 응답: 200/201 등 HTTP 의미에 맞게 사용.
- 에러 응답 구조(예시):

  components:
    schemas:
      Error:
        type: object
        properties:
          code:
            type: string
          message:
            type: string
          details:
            type: object

- 에러 코드 네이밍: `SERVICE_ERROR_CODE` 형식(예: `USER_NOT_FOUND`).

5) 스키마/데이터 규칙
- nullable: OpenAPI 3.1에서는 JSON Schema 방식 사용 (`type: ["string","null"]`).
- 날짜/시간 포맷: RFC 3339 사용 (`format: date-time`).
- ID 필드: 리소스 식별자는 **ULID** 사용을 권장합니다. ULID는 시간 기반으로 정렬 가능하고(URL-safe·URL에 안전하게 포함 가능), 충돌 확률이 낮아 분산 시스템에서 유리합니다. 형식은 문자열이며 OpenAPI 스펙에 `format: ulid`로 표기하거나, 정규식 패턴(예: `^[0-7][0-9A-HJKMNP-TV-Z]{25}$`)을 명시해 검증하세요. 기존 UUID 사용을 원하면 팀 합의로 허용하되, 스펙에 명확히 기술하세요.

6) 엔드포인트 설계 규칙
- 리소스 기반 네이밍: `/users`, `/users/{userId}/orders` 같은 리소스 중심 URI.
- **집합 리소스 복수형**: 컬렉션(복수 리소스)은 반드시 복수형으로 표기합니다. 예: `/users` (O), `/user` (X), `/orders` (O), `/order` (X). 이를 통해 단일 리소스와 집합 리소스를 URI만으로 구분하게 하고, 클라이언트가 기대하는 RESTful 패턴을 따릅니다.
- 컬렉션과 아이템: 컬렉션 `GET /items`, 생성 `POST /items`, 개별 `GET /items/{id}`.
- 쿼리 파라미터: 필터/정렬/페이징 표준화(아래 페이징 규칙 참조).

**URL 작성 규칙 (권장 템플릿)**

팀 표준 URL 템플릿은 다음과 같이 권장합니다:

```
/ {system_name} / {version} / {resource-group} / {resource}
```

설명:
- `system_name`: 프로젝트 또는 시스템 이름(예: `fastexit`).
- `version`: API 버전(예: `v1` 또는 `v1.0`) — URL에 포함하는 형식은 팀 정책에 따름.
- `resource-group`: 일반적으로 서비스 이름; 서비스 내부에서 추가 분류가 필요하면 하위 그룹 이름을 사용.
- `resource`: 리소스의 나머지 경로(개별 리소스나 하위 리소스 경로).

예시:

```
/fastexit/v1.0/admin-api/external-apis/external-api-01
```

권장사항:
- URL 경로는 소문자, 하이픈(`-`) 사용 권장(공백/대문자 사용 금지).
- 버전은 가능한 한 메이저 버전만 표기하는 것을 권장(`v1`)하되, 세부 정책은 팀에서 결정.
- 민감한 정보(예: 토큰, 비밀번호)는 경로에 포함하지 말고 헤더 또는 바디로 전달.
- 가능한 한 RESTful 리소스 네이밍 관점에서 자원과 행위를 분리하세요(예: `/orders/{id}/cancel` 대신 `POST /orders/{id}/cancellations` 등 API 설계 원칙 고려).

7) 페이징/정렬/필터 규약

집합 리소스(컬렉션)에서 사용되는 파라미터 권장 규칙
- 필터 파라미터 (`filter` 형태 또는 개별 필드 필터):
  - 다중값 필터 지원 여부를 명시해야 합니다(스펙에 `multi=true` 같은 표기를 권장).
  - 다중값 필터는 같은 이름의 필터 파라미터를 여러 번 반복해서 보낼 수 있도록 허용합니다. 예: `?status=active&status=pending`.
  - 다중값 필터는 값들 간에 **OR**로 동작합니다(예: status=active OR status=pending). 
  - 서로 다른 필터 이름은 **AND**로 결합됩니다(예: `status=active&type=admin` → status=active AND type=admin).
  - 필터는 기본적으로 **일치 검색(exact match)** 으로 동작하도록 명시하되, 필요 시 `filter_op` 같은 확장 파라미터로 연산자(eq, gt, lt 등)를 허용할 수 있습니다. 스펙에 연산자 지원 여부와 사용법을 명확히 표기하세요.

- 검색 파라미터 (`search_type`, `search_value`):
  - 검색이 필요하면 `search_type`(검색 대상)과 `search_value`(검색어)를 사용합니다. **두 값이 모두 제공되어야 검색이 동작**합니다.
  - `search_type`은 해당 엔드포인트가 가질 수 있는 값들의 목록을 OpenAPI 스펙에 반드시 명시해야 합니다(예: `title`, `description`, `all` 등). 기본값은 `ALL`로 정의할 수 있습니다.
  - 검색 동작은 기본적으로 **부분일치 검색(partial match)** 으로 동작하도록 문서화하세요(예: `contains`, `icontains` 등 구체적 매칭 규칙 표기).
  - 검색 대상이 추가되면 `search_type`에 가능한 값들을 업데이트해야 합니다.

- 페이징 파라미터:
  - 권장: `page` (1-based) 및 `page_size` 또는 `limit`/`offset`(0-based) 사용을 문서화.
  - `page_size`에는 최소/최대값(예: min=1, max=100)을 규정하고 스펙에 명시하세요.

- 정렬 파라미터 (`order_by`):
  - 정렬이 필요하면 `order_by` 파라미터를 사용합니다. 다중 정렬 규칙은 쉼표(,)로 구분합니다. 예: `?order_by=created_at:desc,name:asc`.
  - 정렬 가능한 필드는 스펙에 목록으로 명시하고, 각 필드의 기본 정렬 방향(또는 방향 지정 방식)도 문서화하세요.

- 요청 목적 파라미터 (`purpose`):
  - 하나의 엔드포인트가 서로 다른 목적(예: UI 화면, 내보내기, 관리자 용)으로 호출될 때, `purpose` 쿼리 파라미터를 사용하여 응답을 최적화합니다.
  - 예: `GET /users?purpose=user-selection` (사용자 선택 UI용), `GET /users?purpose=admin-list` (관리자 목록용).
  - `purpose`는 응답 필드 필터링(민감 정보 제외), 조회 범위 제한(팀 단위 제한 등), 감사 로깅을 위한 메타정보 역할을 합니다.
  - **보안 주의**: `purpose` 파라미터는 응답 형식 최적화 힌트일 뿐, 실제 접근 제어는 **사용자 인증 토큰 + 역할(Role) 기반**으로 처리해야 합니다.
  - OpenAPI 스펙에서 `purpose`의 가능한 값을 `enum`으로 명시하세요.

예시 쿼리:
```
GET /items?status=active&status=pending&search_type=title&search_value=fast&page=2&page_size=20&order_by=created_at:desc,name:asc
GET /users?purpose=user-selection&page=1&page_size=50
```

문서화 요구사항
- 각 컬렉션 엔드포인트의 OpenAPI 스펙에 사용 가능한 필터 이름, `search_type` 값 목록, 페이지 사이즈 제한, 정렬 가능한 필드와 형식을 명시하세요.
- 필터/검색/정렬 동작(부분일치 vs 완전일치, 대소문자 구분 등)과 일관성(예: 모든 컬렉션에서 동일한 규칙 적용)을 문서에 명확히 적시하세요.

8) 인증·인가
- 사용자 인증 방식: **Bearer 토큰**을 사용합니다.
  - 형식: `Authorization: Bearer <token>` (HTTP 헤더)
  - 토큰 타입: JWT 또는 opaque 토큰(팀 선택) — 선택한 타입을 OpenAPI 스펙에 명시하세요.
  - 토큰 만료: 적절한 TTL 정책 규정(예: 1시간) — 스펙/문서에 명시해 클라이언트가 refresh 타이밍을 알 수 있게 하세요.
  - Refresh 토큰(선택사항): 장시간 접속이 필요하면 별도 refresh 토큰 엔드포인트(`POST /auth/refresh`) 제공을 고려하세요.
- 공통 스키마로 인증을 정의하고(예: `components/securitySchemes/bearerAuth`), 엔드포인트별 `security` 항목으로 적용합니다.
- 모든 인증이 필요한 요청은 **HTTPS 필수**. 평문 HTTP에서는 토큰 전송 금지.
- 토큰 검증(JWT의 경우 서명 확인, opaque의 경우 서버 저장소 조회) 정책을 문서화하세요.
- 토큰 갱신/폐기 정책(로그아웃, 비활성화)도 문서에 포함하세요.

9) 버전·호환성 정책
- 비호환(파괴적) 변경 시: 메이저 버전 증가 및 별도 엔드포인트 유지 정책.
- 하위 호환 변경은 패치/마이너 버전으로 처리.
- 스펙 변경 시 마이그레이션 가이드와 샘플 요청/응답을 포함.

10) 문서화·샘플
- 각 엔드포인트에 요약(summary), 상세 설명(description), 요청/응답 예시(`example`)을 포함.
- 공통 헤더(예: `X-Request-Id`, `traceparent`) 사용 권장 및 설명.

10-1) 운영 및 추적성 정책 (X-Request-Id, Trace Context)

**X-Request-Id 전파(Propagation) 원칙:**
- 모든 마이크로서비스 및 BFF는 들어오는 요청(Inbound)에 `X-Request-Id` 헤더가 존재하면, 이를 하위 서비스 호출(Outbound) 시 그대로 복사하여 전달해야 합니다.
- 만약 요청에 `X-Request-Id` 헤더가 없다면, 가장 먼저 요청을 받은 진입점(BFF, API Gateway, 또는 서비스 경계)이 새로운 UUID를 생성하여 헤더에 주입합니다.
- 생성된 ID는 이후 모든 구간에서 절대로 변경하거나 새로 생성하지 않고, 전달받은 ID를 그대로 유지하여 전파해야 합니다.
- 목적: 사용자 요청 추적, 로깅 상관관계(correlation), 문제 진단 및 성능 분석에 활용.

**Trace Context (OpenTelemetry):**
- W3C 표준인 `traceparent` 헤더 전파를 지원하는 라이브러리/미들웨어를 사용해야 합니다.
- 분산 추적(Distributed Tracing) 인프라(예: Jaeger, Zipkin, 클라우드 제공자의 관찰성 서비스)와 연동하여 요청 흐름을 가시화하고 성능 병목을 식별합니다.
- OpenTelemetry SDK/자동 계측(auto-instrumentation)을 프레임워크에 통합하여 `traceparent` 헤더 자동 전파를 권장합니다.

**OpenAPI 스펙 표기:**
- 공통 헤더(`X-Request-Id`, `traceparent`)를 `components/parameters` 또는 각 엔드포인트의 `parameters` 섹션에 정의하고, 필수/선택 여부를 명시하세요.
- 예시 헤더 정의:
  ```yaml
  parameters:
    - name: X-Request-Id
      in: header
      required: false
      description: 요청 추적 ID (없으면 서버에서 생성)
      schema:
        type: string
        format: uuid
    - name: traceparent
      in: header
      required: false
      description: W3C Trace Context (분산 추적 활성화 시)
      schema:
        type: string
  ```

10-2) 멱등성(Idempotency) 처리 정책

**POST 요청의 멱등성 보장:**
- 중복 요청에 대한 안전성을 보장하기 위해 `Idempotency-Key` 헤더를 지원해야 합니다.
- 클라이언트는 POST 요청을 보낼 때 `Idempotency-Key` 헤더에 고유한 값(예: UUID, 또는 클라이언트가 생성한 임의의 문자열)을 포함해야 합니다.
- 서버는 동일한 `Idempotency-Key`로 받은 요청에 대해:
  - 첫 번째 요청은 정상적으로 처리하고 결과를 저장합니다.
  - 같은 `Idempotency-Key`로 오는 중복 요청은 저장된 결과를 그대로 반환합니다(재처리 없음).
- `Idempotency-Key`의 유효 기간을 정책으로 설정합니다(예: 24시간, 스펙에 명시).
- 멱등성 저장소(캐시·데이터베이스)의 관리 및 클린업 정책도 운영 가이드에 포함하세요.

**구현 요구사항:**
- 멱등성 키 처리 상태는 반드시 **Redis와 같은 분산 캐시**를 사용하여 공유해야 합니다.
- 다중 인스턴스 환경을 고려하여 **개별 서버의 로컬 메모리 사용은 금지**됩니다.
- 분산 캐시를 통해 모든 인스턴스가 동일한 멱등성 키 상태를 조회하고 갱신할 수 있도록 보장하세요.
- TTL(Time To Live) 설정으로 만료된 키를 자동 삭제하고, 스토리지 용량을 관리하세요.

**사용 예시:**
```
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "items": [...],
  "total": 100
}
```

**OpenAPI 스펙 표기:**
- POST/PUT 요청이 생성/변경 작업을 수행하는 엔드포인트에 `Idempotency-Key` 헤더를 정의하세요.
- 예시:
  ```yaml
  post:
    operationId: createOrder
    parameters:
      - name: Idempotency-Key
        in: header
        required: true
        description: 멱등성 처리를 위한 고유 키 (클라이언트 생성)
        schema:
          type: string
          format: uuid
    responses:
      '201':
        description: Created (또는 이미 존재하는 리소스 반환)
  ```

11) 린트·정책 (CI 통합)
- Spectral 사용 권장: 팀 규칙을 설정해 네이밍·보안·응답 일관성을 자동 검사.
- 스펙 유효성 검사: `openapi-cli` 또는 `openapi-schema-validator`로 기본 문법 검사.
- PR 체크리스트에 다음 항목 포함:
  - Spectral 통과
  - 예제 추가 여부
  - 관련 유스케이스/글로서리 링크 포함

12) 모킹·테스트·코드 생성
- 모킹: Prism 또는 WireMock으로 모킹 서버 제공하여 프론트엔드/QA 병행 개발.
- 코드 생성: OpenAPI Generator 또는 `fastapi-code-generator`로 스텁/클라이언트 생성 가능. 생성물은 수동 검토 필수.

13) 스펙-코드 동기화(드리프트 방지)
- 전략 옵션:
  - 스펙-퍼스트: 스펙을 SSOT로 둔 뒤 스텁/서버를 생성해 구현.
  - 코드-퍼스트: 코드에서 스펙을 생성하고 PR에서 스펙 검증을 의무화.
- 선택한 전략은 팀 규칙으로 고정하고 CI로 강제하세요.

14) 변경 프로세스
- 스펙 변경 PR은 다음을 포함해야 함:
  - 변경 요약
  - 영향받는 유스케이스/용어/서비스 목록
  - Spectral 린트 결과
  - 소비자 영향(호환성 분류: 패치/마이너/메이저)
- 배포 전: 스테이징에서 모킹/통합 테스트 수행.

15) 예시 헤더(간단)

```yaml
openapi: 3.1.0
jsonSchemaDialect: https://json-schema.org/draft/2020-12/schema
info:
  title: Example Service API
  version: 1.0.0
servers:
  - url: https://api.example.com/v1
```

16) 체크리스트(요약)
- `openapi/`에 스펙 존재
- `info.version` 명시(semver)
- Spectral 통과
- 공통 에러 스키마 존재
- 보안 스키마 정의 및 적용
- 샘플 요청/응답 포함
- 관련 유스케이스·glossary 링크 포함

참고: 이 문서는 팀 규율의 출발점입니다. 팀의 요구에 맞게 Spectral 규칙·파일 구조·버전 정책 등을 커스터마이즈하세요.

## 외부 참고 가이드라인

이 표준에서 명시하지 않았거나 더 상세한 사례가 필요한 항목은 아래의 공신력 있는 가이드라인을 참고하세요. 회사 정책과 충돌하는 경우에는 조직 정책을 우선으로 하되, 가능한 모범 사례를 적용하세요.

- **Microsoft REST API Guidelines (GitHub)**: https://github.com/microsoft/api-guidelines
- **Google API Design Guide**: https://cloud.google.com/apis/design

위의 가이드라인은 API 네이밍, 리소스 설계, 에러 핸들링, 보안 권장사항 등 광범위한 베스트 프랙티스를 다룹니다. 필요한 경우 팀 규칙으로 일부 항목을 이 표준에 병합해 사용하세요.
