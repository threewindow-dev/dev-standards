# Python 네이밍 컨벤션 — Repository Model/Entity

## 2.3 Repository Model/Entity (선택 사항)

기본 표준에서는 **인프라 레이어 내부 전용 UserModel 같은 Repository Model을 정의하지 않습니다.**

- 기본 권장: ORM Entity만 사용하고, 바로 도메인 모델(또는 애플리케이션 DTO)로 매핑합니다.
- 프로젝트별 필요 시: 복잡한 프로젝션/다중 스키마/ORM 탈피 등을 위해 인프라 전용 모델을 도입할 수 있으며, 이 경우 규칙을 프로젝트에서 정의합니다.

### 도입이 필요한 경우의 최소 가이드 (선택)
- 위치: `infra/models/` 또는 인프라 하위 네임스페이스
- 네이밍: **도메인 네이밍과 충돌하지 않도록 구분**
	- 도메인 전용 예약: `{Resource}Criteria`, `{Resource}PagedCriteria`, `Create/Update/Delete{Resource}Command`, `{Resource}Dto`, Domain Model `Resource`
	- 인프라 전용 예시(필요 시): `Infra{Resource}Projection`, `Infra{Resource}Row`, `Infra{Resource}QueryResult`
- 외부 노출 금지: 인프라 전용 모델은 애플리케이션·도메인 레이어에 노출하지 않고, 인프라 내부에서 Entity/Raw 결과 ↔ 도메인 모델 사이 변환 용도로만 사용

### 결론
- 표준 범위: ORM Entity만 필수, 인프라 전용 모델은 표준에서 다루지 않음
- 필요 시: 각 프로젝트에서 별도 규칙을 마련하되, 도메인 예약 네이밍(`Criteria`, `PagedCriteria`, Command/Dto 등)과 충돌하지 않게 인프라 접두사/네임스페이스로 분리
