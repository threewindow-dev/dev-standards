# Glossary Template (도메인 공통어)

간단 설명: 핵심 용어를 명확히 정의하여 팀 전체의 공통 이해를 보장합니다.

| Term | Definition | Synonyms | Example | Related Use Cases | API Mapping | Owner |
|------|------------|----------|---------|-------------------|-------------|-------|
| 사용자 (User) | 서비스에 액세스하는 주체. 인증된 개인 또는 시스템. | 고객, account-holder | 사용자 이메일로 로그인 | UC-001 회원가입, UC-002 로그인 | `components/schemas/User` | Product Owner |

작성/변경 지침:
- 소유자(Owner)를 명시하세요.
- 변경 시 변경 이유와 관련 유스케이스를 함께 기록하세요.
- 용어는 API 스펙, 문서, 테스트 케이스에 동일하게 사용하세요.
