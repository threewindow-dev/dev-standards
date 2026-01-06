# 유스케이스 문서 작성 표준

목적: 유스케이스 문서 작성의 일관된 형식과 절차를 제공하여 요구사항·설계·API 스펙 간의 명확한 연결을 보장한다.

핵심 원칙
- 유스케이스는 비즈니스 관점(업무 유스케이스)과 시스템/기능 관점(기능 유스케이스)으로 분리해 작성한다.
- 모든 유스케이스는 `glossary`(도메인 공통어)에서 정의된 용어를 사용해야 한다.
- 유스케이스는 반복적으로 개선되며 변경은 관련 문서(API 스펙·글로서리 등)에 동기화해야 한다.

1. 문서 구조
- 문서 위치: `usecases/` 디렉터리(예: `usecases/business/`, `usecases/functional/`)
- 파일 네이밍: `BC-<번호>-<slug>.md` (업무 유스케이스), `FC-<번호>-<slug>.md` (기능 유스케이스)

2. 업무 유스케이스 (Business Use Case)
- 목적: 사용자의 비즈니스 목표와 시나리오를 서술한다. 기술적 구현은 포함하지 않는다.
- 필수 항목:
  - ID, Title, Summary
  - Actors (이해관계자)
  - Preconditions / Assumptions
  - Main Flow (성공 시나리오)
  - Alternative / Exception Flows
  - Non-functional Requirements (보안, 성능, 규정 등)
  - Acceptance Criteria
  - Related Glossary Terms
  - Related Functional Use Cases

3. 기능 유스케이스 (Functional Use Case)
- 목적: 업무 유스케이스를 시스템 관점으로 분해하여 요구되는 기능·데이터·인터페이스를 정의한다.
- 필수 항목:
  - ID, Title, Related Business Use Case
  - Trigger / Preconditions
  - Inputs / Outputs (데이터 모델 명시)
  - Detailed Steps (시퀀스 또는 상태 흐름)
  - Security / Authorization
  - Performance / Scaling requirements
  - Error Handling (예상 오류와 대응)
  - API Mapping (권장 엔드포인트, 예시 요청/응답)

4. 작성·검토 프로세스
- 초안 작성: 도메인 담당자(PO) 또는 분석가가 업무 유스케이스 초안 작성.
- 기능 전환: 아키텍트/엔지니어가 기능 유스케이스로 분해.
- 리뷰: 관련 팀(도메인, API, QA, 보안)에서 검토 후 승인.
- 승인 기준: Acceptance Criteria 충족, 용어 일관성, 보안·비기능 요구 반영.

5. 변경 관리와 동기화
- 모든 변경은 PR로 제출하고 관련 유스케이스·글로서리·API 스펙을 참조해야 함.
- PR 템플릿에는 "영향받는 문서" 필드를 포함해 변경 범위를 명시.
- CI/PR에서 다음 검증을 권장:
  - glossary 참조 확인(용어 일관성)
  - 관련 API 스펙 존재 여부
  - Spectral 린트(스펙 변경 시)

6. 템플릿 사용
- 팀은 `templates/business_use_case_template.md`와 `templates/functional_use_case_template.md`를 사용해 작성한다.

7. 예시 워크플로(간단)
1. PO가 `BC-001-회원가입.md` 작성(사업 흐름, 요구)
2. 엔지니어가 `FC-001-회원생성.md` 작성(데이터·API 매핑)
3. API 담당자가 `openapi/`에 엔드포인트 초안을 작성
4. 리뷰에서 용어·스펙·보안 점검 후 승인

마무리
- 유스케이스는 설계의 근간입니다. 업무와 기능을 분리해 문서화하고, 용어와 스펙을 항상 동기화하는 것을 규율화하세요.
