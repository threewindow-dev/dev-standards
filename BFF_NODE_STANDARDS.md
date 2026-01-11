# Node.js BFF 표준

## 목적 및 범위
- 프론트엔드 전용 BFF(Backend For Frontend)를 Node.js로 구현할 때의 공통 표준을 정의한다.
- REST/GraphQL 모두 적용 가능하나, 기본 예시는 REST를 기준으로 한다.
- 대상 런타임: Node.js 24 LTS, TypeScript 5.x.
- 기본 권장 프레임워크: Next.js App Router 기반 API Routes. (SSR/CSR + BFF를 단일 런타임에서 처리)
- 대안: Express/Fastify 기반 경량 BFF (SSR 불필요하고 라우팅만 필요할 때).

## 언제 BFF를 사용해야 하는가
- 다중 백엔드(예: 사용자 서비스, 결제 서비스, 검색 서비스)를 단일 프론트엔드 진입점으로 통합해야 할 때.
- 프론트엔드 맞춤 응답(필드 축소, 데이터 가공)이 필요할 때.
- 인증/인가 토큰을 프론트에 노출하지 않고 서버 측에서 안전하게 취급할 때.
- CORS, Rate Limit, 캐싱, 장애 격리(타임아웃/리트라이/폴백)가 필요한 경우.

## 아키텍처 원칙
- 단일 책임: BFF는 **프레젠테이션 계층 어댑터** 역할만 수행한다. 도메인 규칙은 백엔드 서비스로 위임한다.
- 무상태(stateless): 세션 상태는 토큰/쿠키로 관리하고, 서버 상태 저장을 피한다.
- 실패 격리: 백엔드 호출에 타임아웃, 재시도, 폴백(빈 배열/캐시)을 적용한다.
- 최소 데이터 전송: 필요한 필드만 전달하고, 페이징/쿼리 파라미터를 적극 사용한다.
- 보안 기본값: 서버-서버 호출에서만 시크릿을 사용하며, 클라이언트로 노출 금지.

## 기술 스택 가이드
- 런타임: Node.js 24.x LTS (RUNTIME_VERSION_STANDARDS 준수).
- 언어: TypeScript 5.x, `strict` 모드 필수.
- 프레임워크 권장: Next.js App Router (API Routes로 BFF 구현). 대안: Express/Fastify.
- 패키지 관리자: `npm ci` 또는 `pnpm i --frozen-lockfile`.
- HTTP 클라이언트: `fetch`(내장) 우선, 필요 시 `undici`, OAuth 등은 전용 클라이언트 사용.

## 라우팅 & 계약(Contract)
- Public 라우트: `/api/*` (BFF 진입점). 내부 백엔드 URL은 노출 금지.
- 스키마 검증: zod 또는 valibot으로 요청/응답 DTO를 검증하고, 4xx 오류 시 명확한 메시지 반환.
- 에러 규약: `{ error: { code, message, detail? } }` 형태를 기본으로 사용.
- 버저닝: `/api/v1/*` 형태로 버전 명시. 프론트와 동시 배포가 어려우면 뒤에 `/v1` 접미어 라우트를 중복 운영 후 점진 전환.

## 보안
- 인증 위임: 브라우저로 받은 토큰(쿠키/Authorization 헤더)을 BFF에서 검증하거나, 백엔드로 전달할 때 최소 권한 스코프만 전파.
- 시크릿 관리: `.env`/런타임 시크릿(Secret Manager/KMS). 코드/리포지토리에 시크릿 금지.
- CORS: 기본 차단, 필요한 오리진만 화이트리스트. Credentials 사용 시 `Access-Control-Allow-Credentials: true` + 동일 오리진 쿠키 정책 준수.
- 보안 헤더: Next.js 미들웨어 또는 서버 설정으로 HSTS, X-Content-Type-Options, X-Frame-Options, CSP를 적용.

## 신뢰성 & 성능
- 타임아웃: 외부/백엔드 호출 기본 3~5초, 서비스 특성에 따라 조정.
- 재시도: 멱등 GET/HEAD에 한정, 지수 백오프로 2~3회.
- 회로 차단: 특정 백엔드 장애 시 빠른 실패 + 캐시/폴백 활용.
- 캐싱: 브라우저 캐시 제어, CDN 캐시를 염두에 둔 Cache-Control 설정. 서버 캐싱이 필요하면 TTL을 명시하고, 무효화 전략 정의.
- 압축: 응답 gzip/br 수준은 Gateway 또는 CDN에 위임. Next.js는 기본 지원.

## 로깅 & 관찰성
- 구조적 로깅: JSON 형태, `timestamp, level, requestId, route, backend, latencyMs, status` 포함.
- 트레이싱: OpenTelemetry 헤더(`traceparent`, `tracestate`) 전달/생성. 백엔드와 연동 가능하도록 설정.
- 메트릭: 주요 라우트의 요청/성공/에러 카운터, 백엔드 호출 latency 히스토그램.

## 환경별 정책
- 로컬 개발: Gateway 없이 Next.js/Express 단독 실행. `.env.local`로 백엔드 엔드포인트 설정.
- 프로덕션: 앞단에 Gateway/Ingress(Nginx, Kong, ALB 등) 배치. SSL 종료, WAF/Rate Limit, 로드 밸런싱을 Gateway에서 수행.

## 배포
- Next.js: `next build` + `next start` (standalone 모드 권장) 또는 Vercel/Cloud Run 배포.
- Express/Fastify: `npm ci && npm run build && npm run start`로 컨테이너 이미지화. 최소 권한 사용자로 실행.
- 컨테이너: healthcheck 필수, `PORT`와 `HOST=0.0.0.0` 노출. 프로세스 관리자(pm2) 대신 단일 프로세스 + 컨테이너 오케스트레이션 사용.

## 테스트
- 단위 테스트: 핸들러/서비스 레이어에 대해 fast-check(프로퍼티 기반) 또는 Jest/ Vitest 사용.
- 통합 테스트: 백엔드 목킹(HTTP mock) + 스키마 검증 포함. 계약 위반 시 빌드 실패.
- E2E: Playwright/Cypress로 핵심 사용자 플로우 검증. BFF와 백엔드 사이의 실제 호출 경로를 최소 1개 시나리오에서 검증.

## 체크리스트
- [ ] Node 24.x, TS 5.x, strict 모드
- [ ] `/api/*` 라우트, 스키마 검증 적용
- [ ] 백엔드 호출 타임아웃/재시도/회로 차단 설정
- [ ] 인증/시크릿 서버 측 관리, CORS 제한
- [ ] 구조적 로깅 + trace 헤더 전달
- [ ] 로컬: Gateway 없음 / 프로덕션: Gateway 필수
- [ ] CI에서 lint/test/build 통과 후 배포
