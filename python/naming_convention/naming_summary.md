# Python 네이밍 컨벤션 — 요약 & 변경 이력

문서 탐색: [모듈(파일)](naming_modules.md) · [Schema & DTO](naming_schema_dto.md) · [Repository/Entity](naming_repository_model.md) · [Domain Model](naming_domain_model.md) · [요약 & 변경 이력](naming_summary.md) · [트랜잭션 관리](../TRANSACTION_MANAGEMENT.md) · [테스트 전략](../TESTING_STRATEGY.md)

## 요약 테이블

| 레이어 | 접미사 | 단/복수 | 예시 |
|--------|--------|---------|------|
| Models (domain/models) | - (또는 `_model`) | 단수 | `user.py` |
| Criteria (domain/models/criteria) | `_criteria` | 단수 | `user_criteria.py`, `user_paged_criteria.py` |
| ORM Entities (infra/orm/entities) | `_entity` (선택) | 단수 | `user_entity.py` |
| (선택) Infra Models | 프로젝트 자율 (도메인 네이밍과 구분) | 단수 | `InfraUserProjection` 등 |
| Protocols | `_protocol` | 단수 | `user_repository_protocol.py` |
| AppService | `_app_service` | 단수 | `user_app_service.py` |
| IntService | `_int_service` | 단수 | `payment_int_service.py` |
| QueryService | `_query_service` | 단수 | `user_query_service.py` |
| DomainService | `_domain_service` | 단수 | `order_pricing_domain_service.py` |
| Repositories | `_repository` | 단수 | `user_repository.py` |
| DTOs | `_dto` | 단수 | `user_dto.py` |
| Schemas | `_schema` | 단수 | `user_schema.py` |
| Routers | `_router` | 단수 | `user_router.py` |
| Clients | `_client` | 단수 | `payment_client.py` |
 
## 메서드 네이밍 (레이어별)

| 레이어 | 패턴 | 설명 | 예시 |
|--------|------|------|------|
| Interface (Router) | `post_{resource}` / `put_{resource}` / `patch_{resource}` / `delete_{resource}` | HTTP 메서드에 1:1 대응 | `post_user` |
| Interface (Router) | `get_{resource}` / `get_{resource}_list` / `get_{resource}_paged_list` | 조회, 목록, 페이징 | `get_user_paged_list` |
| Application (App/Int/Query Service) | `register_{resource}` / `update_{resource}` / `delete_{resource}` | 상태 변경 유스케이스 | `register_user` |
| Application (App/Query Service) | `get_{resource}` / `get_{resource}_list` / `get_{resource}_paged_list` | 조회 유스케이스 | `get_user_list` |
| Domain (Domain Service / Repository Port) | `create_{entity}` / `update_{entity}` / `delete_{entity}` | 도메인 명령 | `create_user` |
| Domain (Domain Service / Repository Port) | `get_{entity}_by_id` / `query_{entity}` | 조회/검색 | `get_user_by_id`, `query_user` |

### Repository 메서드 추가 규칙
- 존재 여부 확인은 `exists_by_{field}` 메서드 사용 (EXISTS 쿼리로 최적화)
- AppService에서 무결성 검증 시 `get_{entity}_by_id` 대신 `exists_by_{entity}_id` 우선 사용

## 버전 이력
 - 1.10.0 (2026-01-11): 통합 응답 구조 표준 추가 (code/message/data 래핑, HTTP Status + Body code 병행)
 - 1.9.1 (2026-01-11): Repository `exists_by_{field}` 메서드 규칙 추가 (EXISTS 최적화, 무결성 검증)
 - 1.9.0 (2026-01-11): 메서드 네이밍 규칙 추가 (Interface/App/Domain 레이어별 패턴)
