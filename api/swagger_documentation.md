# API Swagger 문서화 표준

## 5.1 스웨거 문서화

### 스웨거 설명 구조
- 스웨거 설명은 다음 순서로 구성:
  1. 기능 설명
  2. 권한 요구사항
  3. 파라미터 분류 (필터, 페이징, 정렬 등)
  4. 요청바디에서 넘어오는 파라미터는 설명에서 제외
  5. 파라미터의 타입은 설명에서 제외
- 응답값은 `response_model`과 `responses` 섹션에서만 정의
- 명확한 섹션 구분 사용 (## 기능, ## 권한, ## 필터 파라미터 등)

### Interface Layer 스웨거 문서화 규칙
- interface 레이어의 Request와 Response의 Field, Path, Query에는 반드시 `examples`를 추가해야 함
- `examples`는 배열 형태로 작성: `examples=["value1", "value2", "value3"]`
- `example`는 deprecated 되었으므로 사용하지 않음
- 각 필드별로 다양한 예시 값을 제공하여 API 사용자의 이해도 향상

### 스웨거 설명 규칙
- 스웨거 설명에서 응답값은 제외하고 `response_model`로 처리
- 파라미터를 목적별로 분류하여 설명 (필터, 페이징, 정렬 등)
- 일관된 문서 구조 사용: 기능 → 권한 → 파라미터 순서

### 공통 응답 규칙
- 공통 에러 응답은 `common_responses` 모듈을 사용하여 일관성 유지
- `from core.common_responses import common_responses`로 import
- `**common_responses`를 사용하여 401, 403, 500 등 공통 에러 응답 포함
- 목적: API 문서의 일관성과 유지보수성 향상

## 5.2 API 파라미터 구성

### 파라미터 분류
- **필터 파라미터**: 검색/필터링 목적 (`company_id`, `company_name` 등)
- **페이징 파라미터**: 페이지네이션 목적 (`offset`, `limit`)
- **정렬 파라미터**: 정렬 목적 (`order_by`)
- 각 파라미터 그룹의 목적을 명시

### API 파라미터 구성 규칙
- API 파라미터를 목적별로 명확히 분류
- 각 파라미터 그룹의 목적을 명시
- 목적: API 사용자에게 명확한 파라미터 이해 제공
