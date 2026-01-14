# SQL 스키마 관리 표준

## 개요

데이터베이스 스키마를 SQL 파일로 관리하여 버전 관리, 추적성, 재사용성을 향상시킵니다.

---

## 폴더 구조

```
backend/
├── sql/
│   ├── schema/           # 테이블 스키마 정의 (DDL)
│   │   ├── 000_users_init.sql
│   │   ├── 001_orders_init.sql
│   │   └── 002_order_items_init.sql
│   ├── migrations/       # 스키마 변경 (향후 확장)
│   └── seeds/            # 초기 데이터 (선택)
└── src/
    └── main.py
```

---

## 파일 명명 규칙

### 스키마 파일 (schema/)

**형식**: `NNN_tablename_description.sql`

**규칙**:
- **NNN**: 3자리 숫자 (000-999)
  - 의존성 순서대로 실행됨
  - 000부터 시작
  - 10 단위로 증가 권장 (중간 추가 여유 확보)
- **tablename**: 테이블명 (소문자, snake_case)
- **description**: 파일 목적 (예: init, indexes, constraints)
- **확장자**: `.sql`

**예시**:
```
000_users_init.sql          # 의존성 없음 (기본 테이블)
010_categories_init.sql     # 의존성 없음
020_orders_init.sql         # users에 의존
030_order_items_init.sql    # orders에 의존
```

---

## SQL 파일 작성 규칙

### 1. 파일 헤더 (필수)

```sql
-- Table name initialization
-- Dependencies: table1, table2 (또는 None)
-- Description: Purpose of this schema file
```

### 2. 테이블 생성

```sql
CREATE TABLE IF NOT EXISTS table_name (
    id SERIAL PRIMARY KEY,
    column1 VARCHAR(100) NOT NULL,
    column2 INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**권장사항**:
- `CREATE TABLE IF NOT EXISTS` 사용 (멱등성 보장)
- Primary Key 명시
- NOT NULL 제약조건 명시
- DEFAULT 값 설정

### 3. 인덱스 생성

```sql
-- Create indexes for performance
CREATE INDEX IF NOT EXISTS idx_table_column ON table_name(column);
CREATE UNIQUE INDEX IF NOT EXISTS idx_table_unique_col ON table_name(unique_col);
```

### 4. 외래 키 (Foreign Key)

```sql
-- Add foreign key constraints
ALTER TABLE order_items 
ADD CONSTRAINT fk_order_items_order_id 
FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;
```

### 5. 샘플 데이터 (선택, 개발 환경만)

```sql
-- Insert sample data (for development only)
-- This section should be removed or conditional in production
INSERT INTO users (username, email, full_name)
SELECT 'john_doe', 'john@example.com', 'John Doe'
WHERE NOT EXISTS (SELECT 1 FROM users WHERE username = 'john_doe');
```

**규칙**:
- `INSERT ... SELECT ... WHERE NOT EXISTS` 패턴 사용 (중복 방지)
- 프로덕션에서는 주석 처리 또는 제거

---

## 실행 순서

### 자동 실행 (main.py)

main.py의 `_initialize_schema_from_sql()` 함수가 다음 순서로 실행:

1. `backend/sql/schema/` 디렉토리 탐색
2. `*.sql` 파일을 파일명 기준 **사전순 정렬**
3. 순서대로 SQL 실행
4. 세미콜론(`;`)으로 구분된 각 명령 개별 실행

**예시 실행 순서**:
```
000_users_init.sql       (1st)
010_categories_init.sql  (2nd)
020_orders_init.sql      (3rd)
030_order_items_init.sql (4th)
```

---

## 의존성 관리

### 원칙

- **독립 테이블**: 000-099 범위
- **1차 의존 테이블**: 100-199 범위
- **2차 의존 테이블**: 200-299 범위

### 예시

```
000_users_init.sql          # 독립
010_categories_init.sql     # 독립
020_products_init.sql       # 독립

100_orders_init.sql         # users(FK)
110_wishlists_init.sql      # users(FK)

200_order_items_init.sql    # orders(FK), products(FK)
210_reviews_init.sql        # users(FK), products(FK)
```

---

## 변경 이력 관리

### 스키마 변경 (향후 마이그레이션 도구 사용 권장)

**현재 방식** (단순한 경우):
- 기존 파일 수정 후 재실행
- `CREATE TABLE IF NOT EXISTS` 덕분에 안전

**권장 방식** (프로덕션):
- 별도 migration 폴더 사용
- Alembic 또는 유사 도구 도입
- 버전별 변경 이력 추적

---

## 예제

### 000_users_init.sql

```sql
-- Users table initialization
-- Dependencies: None
-- Description: Core user management table

CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    full_name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for performance
CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);
CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);

-- Insert sample data (for development only)
INSERT INTO users (username, email, full_name)
SELECT 'john_doe', 'john@example.com', 'John Doe'
WHERE NOT EXISTS (SELECT 1 FROM users WHERE username = 'john_doe');
```

### 100_orders_init.sql

```sql
-- Orders table initialization
-- Dependencies: users
-- Description: User order management

CREATE TABLE IF NOT EXISTS orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending'
);

-- Foreign key
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_user_id 
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;

-- Indexes
CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id);
CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(status);
```

---

## 모범 사례

### ✅ 권장

- 테이블당 하나의 SQL 파일
- 명확한 의존성 순서 (숫자로 표현)
- `IF NOT EXISTS` 사용 (멱등성)
- 외래 키 명시
- 인덱스 생성
- 파일 헤더 작성 (의존성, 설명)

### ❌ 지양

- 여러 테이블을 하나의 파일에 혼합
- 의존성 무시한 순서
- 멱등성 없는 SQL (CREATE TABLE만 사용)
- 외래 키 누락
- 샘플 데이터를 프로덕션에 포함

---

## 버전 이력

- 1.0.0 (2026-01-14): 초기 버전 작성
