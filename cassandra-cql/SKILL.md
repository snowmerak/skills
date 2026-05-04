---
name: cassandra-cql
description: Apache Cassandra의 쿼리 언어 CQL(Cassandra Query Language) 전반을 다룹니다. DDL, DML, 데이터 타입, 인덱싱, Materialized Views, JSON 지원, 보안 등 모든 CQL 관련 작업을 수행할 때 사용합니다. SQL과 유사하지만 Cassandra 특화된 설계 철학을 따릅니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, cql, database, query, ddl, dml]
---

# Apache Cassandra CQL (Cassandra Query Language)

## Overview

CQL은 Cassandra 데이터베이스와 상호작용하는 주요 쿼리 언어입니다. SQL과 유사한 구문을 사용하지만, **분산 데이터베이스 특성의 제약사항**을 이해해야 합니다. CQL 3이 현재 표준이며, 테이블/행/열 개념은 SQL과 동일하게 정의됩니다.

**핵심 원칙**: Cassandra는 RDBMS가 아닙니다. 쿼리 중심 설계가 필수이며, 정규화보다 반정규화가 우선합니다.

---

## SOP: Step-by-Step Procedures

### 1. Keyspace 생성 및 관리

```cql
-- Keyspace 생성 (반복률 설정 필수)
CREATE KEYSPACE IF NOT EXISTS my_app
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,
    'dc2': 3
};

-- 사용 가능한 레플리케이션 클래스 확인
DESCRIBE KEYSPACES;
```

**Step-by-Step**:
1. `NetworkTopologyStrategy` 또는 `SimpleStrategy` 선택 (다중 DC면 전자 필수)
2. 각 데이터센터별 레플리카 수 설정 (일반적으로 3)
3. `IF NOT EXISTS`로 안전성 확보
4. `ALTER KEYSPACE`로 후속 수정 가능

### 2. 테이블 생성 (DDL)

```cql
CREATE TABLE IF NOT EXISTS users (
    user_id UUID,
    email text,
    name text,
    created_at timestamp,
    PRIMARY KEY (user_id)
) WITH comment = 'User profile table';
```

**Step-by-Step**:
1. Partition Key 반드시 정의 (`PRIMARY KEY`)
2. Clustering Column으로 정렬/범위 쿼리 설계
3. `IF NOT EXISTS`로 재실행 안전성 확보
4. 필요한 경우 `WITH` 옵션 설정 (compaction, compression 등)

### 3. 데이터 삽입 및 조회 (DML)

```cql
-- INSERT
INSERT INTO users (user_id, email, name, created_at)
VALUES (uuid(), 'john@example.com', 'John Doe', now());

-- SELECT (Partition Key 필수 포함)
SELECT * FROM users WHERE user_id = <uuid_value>;

-- 범위 조회 (Clustering Column 활용)
SELECT * FROM orders
WHERE customer_id = <uuid>
  AND order_date >= '2024-01-01'
  AND order_date < '2024-02-01';
```

**Step-by-Step**:
1. Partition Key는 WHERE 절에 반드시 포함
2. Clustering Column은 범위 조건으로 활용
3. `ALLOW FILTERING`은 성능 저하 → 사용 금지 (Anti-Pattern)
4. `LIMIT`으로 결과셋 크기 제한 권장

### 4. 인덱싱 전략

```cql
-- Standard Index (단일 컬럼, 소규모 데이터에 적합)
CREATE INDEX ON users (email);

-- SASI Index (Advanced: prefix, order, range 지원)
CREATE CUSTOM INDEX user_email_idx ON users (email)
USING 'org.apache.cassandra.index.sasi.SASIIndex'
WITH OPTIONS = {
    'mode': 'CONTAINS',
    'analyzer': {
        'class': 'org.apache.cassandra.index.sasi.analyzer.StandardAnalyzer'
    }
};

-- SAI Index (5.0+, Storage Attached Index - 권장)
CREATE INDEX ON users (email);  -- 기본 SAI로 자동 생성
```

**Step-by-Step**:
1. Standard Index: 소규모 데이터셋에 사용
2. SASI: 고급 분석 기능 필요 시
3. SAI (5.0+): 신규 프로젝트는 무조건 SAI 권장
4. 인덱스 컬럼의 카디널리티 확인 후 결정

### 5. Materialized Views

```cql
-- 기본 테이블
CREATE TABLE users_by_id (
    user_id UUID PRIMARY KEY,
    email text,
    name text
);

-- Materialized View 생성 (별도 테이블로 자동 유지)
CREATE MATERIALIZED VIEW users_by_email AS
SELECT * FROM users_by_id
WHERE email IS NOT NULL AND user_id IS NOT NULL
PRIMARY KEY (email, user_id);
```

**Step-by-Step**:
1. MV는 별도 저장 공간 사용 → 용량 고려
2. `WHERE` 절에 PK 포함 필수
3. 쓰기 성능 저하 발생 (자동 동기화 오버헤드)
4. 읽기 전용 워크로드에 적합

### 6. JSON 지원

```cql
-- JSON으로 INSERT
INSERT INTO users (user_id, data_json)
VALUES (uuid(), '{"name": "John", "age": 30}');

-- JSON 쿼리
SELECT * FROM users WHERE data_json->>'name' = 'John';
```

### 7. 보안 (RBAC)

```cql
-- 사용자 생성
CREATE USER IF NOT EXISTS app_user WITH PASSWORD 'secure_password';

-- 권한 부여
GRANT SELECT, INSERT ON KEYSPACE my_app TO app_user;
GRANT ALL ON KEYSPACE my_app TO admin_user;

-- 권한 확인
LIST PERMISSIONS OF app_user;
```

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | cqlsh 실행, 스크립트 처리 | `cqlsh -e "DESCRIBE KEYSPACES;"` |
| `search_files` | CQL 쿼리 패턴 검색 | `grep -r "CREATE TABLE" *.sql` |
| `read_file` | CQL 스크립트 읽기/분석 | migration 파일 확인 |
| `edit_file` | DDL/DML 수정 | 테이블 구조 변경 |

**cqlsh 사용법**:
```bash
# 연결
cqlsh <host> <port> -u <username> -p <password>

# 스크립트 실행
cqlsh -f migration.sql

# 출력 형식 지정
cqlsh --format=csv -e "SELECT * FROM users LIMIT 10;"
```

---

## Anti-Patterns & Guardrails

❌ **Partition Key 없이 WHERE 절 사용** — 전체 스캔 발생, 성능 치명적  
❌ **`ALLOW FILTERING` 남용** — 분산 환경에서 전체 노드 스캔 유발  
❌ **RDBMS 정규화 패턴 적용** — Cassandra는 반정규화 필수  
❌ **동일 Partition에 과도한 쓰기** — Hotspot 발생, 레플리카 과부하  
❌ **Standard Index 대용량 컬럼 사용** — 메모리 오버헤드 급증  
❌ **Materialized Views 남발** — 쓰기 성능 저하 + 저장 공간 2배  

⚠️ **Clustering Column 순서 변경 불가** — ALTER TABLE로 변경 불가, 테이블 재생성 필요  
⚠️ **`now()` 함수는 서버 시간 기준** — 클라이언트 시간과 불일치 가능  
⚠️ **TTL은 행/컬럼 단위만 지원** — 쿼리 레벨 TTL 불가  

---

## Best Practices

1. **쿼리 먼저 설계, 테이블 그 다음** — Cassandra의 핵심 철학
2. **Partition Key는 고카디널리티** — 데이터 분산을 위해 필수
3. **Clustering Column은 정렬/범위 목적에 맞게** — ORDER BY와 일치시켜야 함
4. **Batch 사용 최소화** — `UNLOGGED BATCH`만 허용, `LOGGED BATCH`는 성능 저하
5. **SAI (Storage Attached Index) 우선** — 5.0+ 프로젝트는 무조건 SAI
6. **NetworkTopologyStrategy 다중 DC 환경에서 필수**
7. **CQL 스크립트는 버전 관리** — Flyway/Liquibase와 통합 가능

---

## References

- [CQL Definitions](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/definitions.html)
- [Data Types](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/types.html)
- [DDL Reference](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/ddl.html)
- [DML Reference](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html)
- [Indexing Concepts](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/indexing/indexing-concepts.html)
- [Materialized Views](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/mvs.html)
- [CQL Security](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/security.html)
- [JSON Support](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/json.html)
