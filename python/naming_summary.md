# Python 네이밍 컨벤션 — 요약 & 변경 이력

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

## 버전 이력
- 1.8.3 (2026-01-11): 도메인 예약 네이밍(Criteria/Command/Dto)과 인프라 전용 모델 네이밍 충돌 방지 가이드 추가
- 1.8.2 (2026-01-11): 인프라 전용 Repository Model(UserModel 등) 표준을 제거하고, 프로젝트 자율 항목으로 분리
- 1.8.1 (2026-01-11): QueryService 네이밍 규칙 추가 (application/services/*_query_service.py)
- 1.8.0 (2026-01-11): 문서를 주제별 파일로 분리 (modules, schema/dto, repository model, domain model, summary)
- 1.7.2 (2026-01-11): UserCriteria와 UserPagedCriteria로 분리, Schema/DTO 패턴과 일관성 확보, 상속 패턴으로 DRY 유지
- 1.7.1 (2026-01-11): UserQueryCriteria → UserCriteria 단순화, search/filter/paging/order 필드 명확화
- 1.7.0 (2026-01-11): 모듈 네이밍을 Models & Criteria로 정리, ORM Entity 위치를 infra/orm으로 명확화, QueryCondition → QueryCriteria 일관화
- 1.6.0 (2026-01-11): Domain Model 네이밍 규칙 추가 (2.4) - DDD 엄격 적용 시 비즈니스 로직 모델 분리
- 1.5.0 (2026-01-11): Entity 네이밍 규칙 추가 (2.3) - 데이터 계층 모델 패턴 (Create/Update/Delete Model 분리)
- 1.4.0 (2026-01-11): DTO 클래스 네이밍 컨벤션 추가 - CQRS 패턴 (Command/Query) 기반
- 1.3.1 (2026-01-11): ListResponse/PagedListResponse 필드명을 items로 통일 (완전한 일관성 확보)
- 1.3.0 (2026-01-11): 대량 작업(Bulk Operations) 패턴 추가 - URL 액션 기반 네이밍
- 1.2.0 (2026-01-11): GET 패턴에도 Get 접두사 추가 (모든 HTTP 메서드 완전 일관성 확보)
- 1.1.0 (2026-01-11): Schema 클래스 네이밍 컨벤션 추가 (HTTP 메서드별 Request/Response 패턴)
- 1.0.1 (2026-01-11): Router 네이밍을 복수형에서 단수형으로 변경 (전체 레이어 일관성 확보)
- 1.0.0 (2026-01-11): 초기 버전 작성 — 모듈(파일) 네이밍 컨벤션 정의
