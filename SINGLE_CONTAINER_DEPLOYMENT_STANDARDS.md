# 싱글 루트 워크스페이스 배포 표준

## 목적 및 범위
- 프론트엔드, BFF, 백엔드를 **단일 Git 리포지토리(monorepo)** + **단일 Docker 컨테이너**로 배포하는 표준을 정의한다.
- 프로세스 관리자로 **supervisord**를 사용하여 여러 서비스(PostgreSQL, Backend, Frontend)를 하나의 이미지에서 관리한다.
- 대상 환경: 소규모~중규모 애플리케이션, 개발/스테이징, 간단한 프로덕션 배포.
- 대규모 환경이나 독립적 스케일링이 필요하면 Kubernetes/docker-compose 구조로 전환.

## 언제 사용해야 하는가

### ✅ 적합한 경우
- 모든 서비스가 동시에 배포/업데이트되는 모놀리식 애플리케이션.
- 서비스 간 네트워크 오버헤드를 최소화해야 할 때 (localhost 통신).
- 배포 복잡도 최소화 (이미지 1개, 컨테이너 1개).
- 개발 환경과 프로덕션 환경 일치성 중요 (동일 이미지).

### ❌ 부적합한 경우
- 서비스별 독립적 스케일링이 필요한 경우.
- 서비스별로 다른 배포/업데이트 주기.
- 한 서비스의 장애가 전체에 영향을 미치면 안 되는 경우.
- 이미지 크기가 매우 중요한 경우 (ECR 비용, 배포 속도).

## 아키텍처

### 단일 컨테이너 구조
```
┌──────────────────────────────────────┐
│     Docker Container                 │
├──────────────────────────────────────┤
│   Supervisord (PID 1)                │
│   ├─ PostgreSQL 17 (localhost:5432)  │
│   ├─ FastAPI (localhost:8000)        │
│   └─ Next.js (localhost:3000)        │
└──────────────────────────────────────┘
```

### 프로세스 관리 흐름
```
supervisord 시작
  ├─ PostgreSQL 초기화 (priority=10) → 데이터 로드
  ├─ Backend 시작 (priority=20) → DB 연결 확인
  └─ Frontend 시작 (priority=30) → Backend API 호출
```

각 프로세스는 `autorestart=true`로 설정되어 장애 시 자동 재시작.

## 포트 매핑 및 환경 변수

### 포트 구성
| 서비스 | 컨테이너 포트 | 호스트 포트 | 프로토콜 |
|--------|---------------|-------------|---------|
| Frontend | 3000 | 3000 | HTTP |
| Backend | 8000 | 8000 | HTTP |
| Database | 5432 | 5432 | TCP |

**호스트 포트는 배포 환경에 따라 조정 가능** (예: 프로덕션은 3001, 8001, 5433).

### 필수 환경 변수
```bash
# PostgreSQL
POSTGRES_USER=postgres              # DB 사용자 (기본값: postgres)
POSTGRES_PASSWORD=changeme          # DB 암호 (필수!)
POSTGRES_DB=fastexit                # DB 이름 (기본값: fastexit)

# Backend (FastAPI)
DB_HOST=localhost                   # DB 호스트 (고정)
DB_PORT=5432                        # DB 포트 (고정)
DB_NAME=${POSTGRES_DB}              # 위와 동일
DB_USER=${POSTGRES_USER}            # 위와 동일
DB_PASSWORD=${POSTGRES_PASSWORD}    # 위와 동일

# Frontend (Next.js)
BACKEND_URL=http://localhost:8000   # Backend API URL (고정)
NODE_ENV=production                 # Node 환경 (프로덕션)
PORT=3000                           # 서버 포트 (고정)
HOSTNAME=0.0.0.0                    # 바인드 주소 (고정)
```

## Dockerfile 표준 구조

### 멀티 스테이지 빌드
```dockerfile
# Stage 1: Base (모든 런타임 설치)
FROM python:3.13-bookworm AS base
RUN apt-get update && apt-get install -y \
    nodejs \
    postgresql-17 \
    supervisor \
    ...

# Stage 2: Frontend Build
FROM base AS frontend-builder
COPY frontend/ .
RUN npm ci && npm run build

# Stage 3: Backend Build (필요 시)
FROM base AS backend-builder
COPY backend/ .
RUN pip install -r requirements.txt

# Stage 4: Final (실행 이미지)
FROM base AS runner
COPY --from=frontend-builder /app/frontend/.next /app/frontend/.next
COPY backend/ /app/backend/
COPY deployment/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
CMD ["supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### 이미지 크기 최적화
- 빌드 의존성은 빌드 스테이지에만 포함.
- 최종 스테이지에서만 런타임 필수 패키지 포함.
- `apt-get` 뒤에 `rm -rf /var/lib/apt/lists/*` 추가.
- `.dockerignore` 활용으로 불필요한 파일 제외.

## Supervisord 설정 표준

### supervisord.conf 구조
```ini
[supervisord]
nodaemon=true           # 포그라운드 실행 (Docker PID 1)
user=root               # 루트로 실행
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid

[program:postgresql]
command=...             # PostgreSQL 시작 명령
priority=10             # 첫 번째 시작
autostart=true          # 컨테이너 시작 시 자동 시작
autorestart=true        # 프로세스 종료 시 자동 재시작
startsecs=5             # 5초 이상 실행 시 정상으로 간주
stopwaitsecs=10         # 종료 신호 후 10초 대기

[program:backend]
command=...
priority=20             # 두 번째 시작 (DB 필요)
...

[program:frontend]
command=...
priority=30             # 세 번째 시작 (Backend API 필요)
...
```

### 프로세스 우선순위(priority) 규칙
- **PostgreSQL**: 10 (최우선)
- **Backend**: 20 (DB 초기화 후)
- **Frontend**: 30 (Backend 준비 후)

각 프로세스는 시작 후 `startsecs` 동안 실행되어야 정상으로 간주.

### 로그 관리
```ini
stderr_logfile=/var/log/supervisor/{service}.err.log
stdout_logfile=/var/log/supervisor/{service}.out.log
```

**컨테이너 내부 로그는 Docker 표준 출력(`docker logs`)으로 아직 수집되지 않음.**
→ 프로덕션에서는 Filebeat/Fluentd로 로그 수집 권장.

## 빌드 및 실행

### 빌드 스크립트 (build-single-image.sh)
```bash
#!/bin/bash
IMAGE_NAME=${IMAGE_NAME:-fastexit-monolith}
IMAGE_TAG=${IMAGE_TAG:-latest}

docker build -t ${IMAGE_NAME}:${IMAGE_TAG} \
  -f Dockerfile \
  --build-arg NODE_ENV=production \
  .
```

### 실행 스크립트 (run-single-container.sh)
```bash
#!/bin/bash
CONTAINER_NAME=${CONTAINER_NAME:-fastexit}
IMAGE_NAME=${IMAGE_NAME:-fastexit-monolith}
IMAGE_TAG=${IMAGE_TAG:-latest}
POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-changeme}

docker run -d \
  --name ${CONTAINER_NAME} \
  --restart unless-stopped \
  -p 3000:3000 \
  -p 8000:8000 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=${POSTGRES_PASSWORD} \
  -v ${CONTAINER_NAME}-data:/var/lib/postgresql/data \
  ${IMAGE_NAME}:${IMAGE_TAG}
```

## 데이터 영속성

### PostgreSQL 데이터 저장
```bash
# Docker 볼륨으로 관리 (권장)
docker run -v fastexit-data:/var/lib/postgresql/data ...

# 볼륨 확인
docker volume ls | grep fastexit-data

# 볼륨 백업
docker run --rm \
  -v fastexit-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/fastexit-backup.tar.gz -C /data .

# 볼륨 복원
docker run --rm \
  -v fastexit-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/fastexit-backup.tar.gz -C /data
```

### 주의사항
- 데이터 손실 방지: 항상 백업 수행 후 컨테이너 업데이트.
- 마이그레이션 시: 데이터베이스 스키마 마이그레이션 스크립트 포함.

## 헬스 체크

### 컨테이너 내부 검증
```bash
# 모든 프로세스 상태 확인
docker exec ${CONTAINER_NAME} supervisorctl status

# 특정 프로세스 로그
docker exec ${CONTAINER_NAME} tail -f /var/log/supervisor/backend.out.log

# PostgreSQL 연결 확인
docker exec ${CONTAINER_NAME} psql -U postgres -d fastexit -c "SELECT 1;"

# Backend API 확인
docker exec ${CONTAINER_NAME} curl http://localhost:8000/docs

# Frontend 확인
docker exec ${CONTAINER_NAME} curl http://localhost:3000
```

### 헬스 체크 엔드포인트 (Dockerfile에 추가 권장)
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000 || exit 1
```

## 배포 전략

### 개발 환경
```bash
# 포트 충돌 피하기
docker run -d \
  --name fastexit-dev \
  -p 3001:3000 \
  -p 8001:8000 \
  -p 5433:5432 \
  -e POSTGRES_PASSWORD=dev_password \
  fastexit-monolith:latest
```

### 스테이징 환경
```bash
# 메모리/CPU 제한
docker run -d \
  --name fastexit-staging \
  --memory="1g" \
  --cpus="1" \
  -p 3000:3000 \
  -p 8000:8000 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=${SECURE_PASSWORD} \
  -v fastexit-staging-data:/var/lib/postgresql/data \
  fastexit-monolith:staging
```

### 프로덕션 환경
```bash
# 보안 및 안정성 강화
docker run -d \
  --name fastexit-prod \
  --restart unless-stopped \
  --memory="2g" \
  --cpus="2" \
  -p 3000:3000 \
  -p 8000:8000 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=${SECURE_PASSWORD_FROM_SECRET_MANAGER} \
  -e NODE_ENV=production \
  -v fastexit-prod-data:/var/lib/postgresql/data \
  --log-driver=json-file \
  --log-opt=max-size=10m \
  --log-opt=max-file=3 \
  fastexit-monolith:v1.0.0
```

## 마이그레이션 및 업데이트

### 데이터베이스 마이그레이션
```bash
# 1. 기존 데이터 백업
docker exec fastexit pg_dump -U postgres fastexit > backup.sql

# 2. 새 이미지로 업데이트
docker stop fastexit
docker rm fastexit
docker build -t fastexit-monolith:v1.0.0 .
docker run -d --name fastexit ... fastexit-monolith:v1.0.0

# 3. 마이그레이션 스크립트 실행 (필요 시)
docker exec fastexit psql -U postgres -d fastexit -f /migration/001-schema.sql
```

### 무중단 배포 (프로덕션)
```bash
# 1. 블루-그린 배포: 새 컨테이너에서 실행
docker run -d --name fastexit-green ... fastexit-monolith:v1.0.1

# 2. 헬스 체크 후 트래픽 전환 (Gateway/LB에서)
sleep 30
curl http://localhost:3001/health  # 확인

# 3. 기존 컨테이너 제거
docker stop fastexit-blue
docker rm fastexit-blue
docker rename fastexit-green fastexit-blue
```

## 트러블슈팅

### 프로세스가 시작되지 않음
```bash
# 1. supervisord 상태 확인
docker exec fastexit supervisorctl status

# 2. 프로세스별 로그 확인
docker exec fastexit cat /var/log/supervisor/{service}.err.log

# 3. 프로세스 재시작
docker exec fastexit supervisorctl restart {service}
```

### Database 연결 오류
```bash
# PostgreSQL이 준비되지 않았을 수 있음
# supervisord.conf에서 priority와 startsecs 확인

# 강제 재시작
docker exec fastexit supervisorctl restart all

# 또는 컨테이너 재시작
docker restart fastexit
```

### 높은 메모리 사용
```bash
# 각 서비스 리소스 모니터링
docker stats fastexit

# 필요 시 메모리 제한 조정
docker update --memory="3g" fastexit
```

## 파일 구조

```
fastexit-simple/
├── Dockerfile                          # 단일 컨테이너 이미지 정의
├── .dockerignore                       # Docker 빌드 제외 파일
├── deployment/
│   ├── supervisord.conf                # supervisord 프로세스 관리 설정
│   ├── init-db.sh                      # PostgreSQL 초기화 스크립트
│   └── start.sh                        # 컨테이너 시작 (선택적)
├── scripts/
│   ├── build-single-image.sh           # 이미지 빌드 자동화
│   └── run-single-container.sh         # 컨테이너 실행 자동화
├── backend/
│   ├── src/main.py
│   └── requirements.txt
├── frontend/
│   ├── src/
│   ├── next.config.ts
│   └── package.json
└── README.md
```

## 체크리스트

- [ ] Dockerfile 멀티 스테이지 빌드 구현
- [ ] supervisord.conf 프로세스 우선순위(priority) 설정
- [ ] 환경 변수 필수/선택 분류 및 기본값 설정
- [ ] 로그 파일 위치 정의 + 수집 전략 수립
- [ ] HEALTHCHECK 엔드포인트 구현
- [ ] 데이터 백업/복구 스크립트 작성
- [ ] 개발/스테이징/프로덕션 배포 스크립트 분리
- [ ] 마이그레이션 전략 문서화
- [ ] 모니터링/로깅 설정 (Prometheus, ELK, Datadog 등)

## 참고 및 한계

- **supervisord 대안**: systemd, runit, openrc (호스트 기반), 또는 각 서비스를 독립 컨테이너로 관리.
- **이미지 크기 증대**: 단일 이미지에 모든 의존성 포함으로 이미지 크기가 커짐. 필요 시 멀티 스테이지 빌드로 최소화.
- **확장성 제한**: 프로덕션 스케일링이 필요하면 Kubernetes/docker-compose 전환 권장.
