# 런타임 및 프레임워크 버전 선택 표준

## 목적
- 프로젝트에서 사용하는 런타임(Node.js, Python, Java) 및 주요 프레임워크 버전을 선택하는 일관된 기준을 제공합니다.
- 안정성과 최신성의 균형을 유지하여 보안 업데이트와 새로운 기능을 적절히 활용합니다.

## 적용 대상
- threewindow-dev 조직의 모든 개발 프로젝트

## 런타임 버전 선택 기준

### Node.js
- **기준**: LTS (Long Term Support) 버전 사용
- **이유**: 
  - 장기 지원으로 안정성과 보안 업데이트 보장
  - 프로덕션 환경에 적합한 검증된 버전
  - 대부분의 라이브러리와 호환성 보장
- **확인 방법**: [Node.js 공식 사이트](https://nodejs.org/)에서 "LTS" 라벨이 붙은 버전 확인
- **예시**:
  - Node.js 24.x (LTS) ✅
  - Node.js 25.x (Current) ❌

### Java
- **기준**: LTS (Long Term Support) 버전 사용
- **이유**:
  - 장기 지원으로 안정성과 보안 업데이트 보장
  - 엔터프라이즈 환경에 적합한 검증된 버전
  - 주요 프레임워크와 라이브러리의 공식 지원
- **LTS 버전**: Java 8, 11, 17, 21, 25 등
- **권장**: 가능한 최신 LTS 버전 사용 (현재 기준 Java 25, 2025년 릴리즈)
- **예시**:
  - Java 25 (LTS) ✅
  - Java 21 (LTS) ✅
  - Java 26 (non-LTS) ❌

### Python
- **기준**: 두 번째 버그 패치가 릴리즈된 최신 마이너 버전 사용
- **규칙**:
  - 최신 버전의 마지막 패치 번호가 2 이상이면 해당 마이너 버전 사용
  - 최신 버전의 마지막 패치 번호가 1 이하이면 이전 마이너 버전 사용
- **이유**:
  - 새로운 마이너 버전의 초기 버그가 충분히 수정된 후 사용
  - 최신 기능을 활용하면서도 안정성 확보
  - Python 커뮤니티의 일반적인 관행과 부합
- **예시**:
  - 현재 최신: Python 3.14.2 → **3.14** 사용 ✅ (두 번째 패치 릴리즈 완료)
  - 현재 최신: Python 3.14.1 → **3.13** 사용 ✅ (아직 첫 번째 패치만 릴리즈)
  - 현재 최신: Python 3.14.0 → **3.13** 사용 ✅ (패치 릴리즈 없음)

### PostgreSQL
- **기준**: 두 번째 버그 패치가 릴리즈된 최신 메이저 버전 사용
- **규칙**:
  - 최신 버전의 마지막 패치 번호가 2 이상이면 해당 메이저 버전 사용
  - 최신 버전의 마지막 패치 번호가 1 이하이면 이전 메이저 버전 사용
- **이유**:
  - 새로운 메이저 버전의 초기 버그가 충분히 수정된 후 사용
  - 최신 기능과 성능 개선을 활용하면서도 안정성 확보
  - 프로덕션 환경에서 안전한 데이터베이스 운영
- **참고**: PostgreSQL은 매년 새로운 메이저 버전을 릴리즈하며, 각 메이저 버전은 5년간 지원
- **예시**:
  - 현재 최신: PostgreSQL 18.1 → **17** 사용 ✅ (아직 첫 번째 패치만 릴리즈)
  - 현재 최신: PostgreSQL 18.2 → **18** 사용 ✅ (두 번째 패치 릴리즈 완료)
  - 현재 최신: PostgreSQL 18.0 → **17** 사용 ✅ (패치 릴리즈 없음)

## 프레임워크 버전 선택 가이드라인

프레임워크와 라이브러리는 런타임보다 빠른 릴리즈 주기와 프로젝트별 다른 요구사항으로 인해 엄격한 표준 대신 가이드라인을 제공합니다.

### Python FastAPI
- **권장 방침**: 최신 stable 버전 사용
- **가이드라인**:
  - 새 프로젝트: 최신 메이저 버전 사용 (현재 0.127.x 권장)
  - 기존 프로젝트: 마이너 버전 업데이트는 정기적으로 검토, 메이저 버전은 Breaking Changes 확인 후 결정
  - 보안 패치: 즉시 적용
- **이유**:
  - FastAPI는 활발히 개발 중이며 성능 개선과 새 기능이 자주 추가됨
  - Python 3.13과의 호환성을 위해 최신 버전 유지 필요
  - 타입 힌트와 Pydantic 업데이트를 최대한 활용
- **참고**: [FastAPI Release Notes](https://fastapi.tiangolo.com/release-notes/)

### Node.js React
- **권장 방침**: 최신 stable 메이저 버전 사용
- **가이드라인**:
  - 새 프로젝트: React 19 사용 (최신 Concurrent Features 및 성능 개선)
  - 기존 프로젝트: React 18 → 19 마이그레이션 가이드 참고하여 단계적 업그레이드
  - 보안 패치: 즉시 적용
  - TypeScript 사용 권장 (타입 안정성)
- **이유**:
  - React 18의 Concurrent Rendering, Automatic Batching 등 성능 개선 기능 활용
  - 최신 생태계(Next.js, Remix 등)와의 호환성
  - 커뮤니티 지원과 라이브러리 호환성
- **메이저 버전 업그레이드**: Breaking Changes가 크지 않으므로 적극 권장
- **참고**: [React Releases](https://react.dev/blog)

### Node.js Next.js
- **권장 방침**: 최신 stable 메이저 버전 사용
- **가이드라인**:
  - 새 프로젝트: Next.js 15 사용 (App Router, Server Components 최적화)
  - 기존 프로젝트: Pages Router → App Router 마이그레이션 고려
  - Turbopack 활용 권장 (빌드 성능 개선)
- **이유**:
  - React Server Components 완전 지원
  - 향상된 캐싱 및 성능 최적화
  - Turbopack을 통한 빠른 개발 환경
  - Vercel 최적화 및 엣지 런타임 지원
- **메이저 버전 업그레이드**:
  - 14 → 15는 주요 API 변경 포함
  - 마이그레이션 가이드 참고 필수
- **참고**: [Next.js Releases](https://nextjs.org/blog)

### Java Spring Framework
- **권장 방침**: Spring Boot 최신 stable 버전 사용
- **가이드라인**:
  - 새 프로젝트: Spring Boot 3.x 사용 (Java 17+ 필수, Java 25 권장)
  - 기존 프로젝트: Spring Boot 2.x는 2025년 말 지원 종료 예정, 마이그레이션 계획 수립
  - 보안 패치: 즉시 적용 (Spring Security 관련 특히 중요)
  - Spring Cloud 사용 시 호환성 매트릭스 확인
- **이유**:
  - Spring Boot 3.x는 네이티브 이미지, 관찰성(Observability) 개선 등 현대적 기능 제공
  - Jakarta EE 전환 완료로 장기적 안정성 확보
  - GraalVM Native Image 지원으로 성능 최적화 가능
- **메이저 버전 업그레이드**: 
  - 2.x → 3.x는 Breaking Changes 존재 (javax.* → jakarta.*)
  - 충분한 테스트와 단계적 마이그레이션 필요
- **참고**: [Spring Boot Support](https://spring.io/projects/spring-boot#support)

### 공통 원칙
1. **보안 최우선**: 보안 취약점 발견 시 즉시 패치 버전 업데이트
2. **의존성 관리**: Dependabot, Renovate 등 자동화 도구 활용 권장
3. **테스트 커버리지**: 버전 업그레이드 전 충분한 테스트 작성
4. **단계적 적용**: 개발 → 스테이징 → 프로덕션 순으로 검증 후 배포

## 런타임 버전 업데이트 주기

### 정기 검토
- **주기**: 분기별 (3개월)
- **검토 항목**:
  - 새로운 LTS 버전 릴리즈 여부
  - 현재 사용 중인 버전의 지원 종료 일정
  - 주요 보안 패치 및 버그 수정 사항

### 즉시 업데이트 대상
- 중요 보안 취약점 수정 버전
- 현재 버전의 지원 종료 임박 (3개월 이내)

### 업데이트 프로세스
1. 새 버전 검토 및 호환성 테스트
2. 개발 환경에서 검증
3. 스테이징 환경 배포 및 테스트
4. 프로덕션 환경 단계적 적용

## 프로젝트 문서화

### 필수 사항

각 프로젝트는 다음 두 가지를 반드시 작성해야 합니다:

#### 1. README.md에 버전 정보 명시

```markdown
## 개발 환경

### 필수 런타임
- Node.js: 24.x (LTS)
- Python: 3.13
- Java: 25 (LTS)

### 데이터베이스
- PostgreSQL: 17

### 주요 프레임워크
- FastAPI: 0.127.x (최신 stable)
- React: 19.x
- Next.js: 15.x
- Spring Boot: 3.4.x
```

#### 2. .tool-versions 파일 작성

[asdf](https://asdf-vm.com/) 버전 관리 도구를 위한 `.tool-versions` 파일을 작성합니다.

**파일 위치:**
- **싱글 레포지토리 프로젝트**: 프로젝트 루트에 `.tool-versions` 작성
  ```
  {project-name}-simple/
  ├── .tool-versions          # 여기에 작성
  ├── README.md
  └── ...
  ```

- **멀티 레포지토리 프로젝트**: 각 서비스/리포지토리 루트에 `.tool-versions` 작성
  ```
  {project-name}/
  ├── gateway-service/
  │   ├── .tool-versions      # 각 서비스마다 작성
  │   └── ...
  ├── auth-service/
  │   ├── .tool-versions      # 각 서비스마다 작성
  │   └── ...
  └── payment-service/
      ├── .tool-versions      # 각 서비스마다 작성
      └── ...
  ```

**파일 형식 예시:**

```
# .tool-versions
nodejs 24.11.0
python 3.13.1
java temurin-25.0.1+10.0.LTS
postgres 17.2
```

**작성 규칙:**
- 사용하는 런타임과 도구만 포함
- 정확한 패치 버전까지 명시 (예: `24.11.0`, `3.13.1`)
- Java는 배포판 명시 권장 (예: `temurin-25.0.1+10.0.LTS`)
- 주석을 사용하여 특정 버전 선택 이유 설명 가능

## 참고 자료

### 런타임
- [Node.js Release Schedule](https://github.com/nodejs/release#release-schedule)
- [Python Release Schedule](https://peps.python.org/pep-0602/)
- [Java Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)
- [PostgreSQL Versioning Policy](https://www.postgresql.org/support/versioning/)

### 프레임워크
- [FastAPI Release Notes](https://fastapi.tiangolo.com/release-notes/)
- [React Blog & Releases](https://react.dev/blog)
- [Spring Boot Support](https://spring.io/projects/spring-boot#support)
