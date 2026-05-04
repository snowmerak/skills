---
name: cassandra-data-modeling
description: Apache Cassandra 특화 데이터 모델링을 다룹니다. RDBMS vs NoSQL 설계 차이, 쿼리 중심 접근법, Conceptual/Logical/Physical 설계 단계, Partition Key 및 Clustering Column 전략, 반정규화 패턴 등을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, data-modeling, nosql, schema-design, partition-key]
---

# Apache Cassandra Data Modeling

## Overview

Cassandra 데이터 모델링은 RDBMS와 근본적으로 다릅니다. **쿼리 패턴을 먼저 정의**하고, 그에 맞게 테이블을 설계해야 합니다. 정규화보다 반정규화가 우선이며, 읽기 성능을 최우선으로 고려합니다.

**핵심 철학**: "Query-first design" — 쿼리를 먼저 생각하고, 테이블은 그 다음에 설계합니다.

---

## SOP: Step-by-Step Procedures

### 1. Conceptual Design (개념적 설계)

```
사용자 요구사항 → 쿼리 패턴 분석 → 엔티티 식별
```

**Step-by-Step**:
1. 애플리케이션의 모든 읽기 쿼리 목록 작성
2. 각 쿼리의 필터 조건, 정렬 기준, 조인 필요성 파악
3. 데이터 접근 빈도 및 크기 추정
4. 일관성 요구사항 확인 (Strong vs Eventual)

### 2. Logical Design (논리적 설계)

```cql
-- 예시: 주문 시스템
CREATE TABLE orders_by_customer (
    customer_id UUID,
    order_id UUID,
    order_date timestamp,
    total_amount decimal,
    status text,
    items list<text>,
    PRIMARY KEY (customer_id, order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC);

-- 별도 테이블: 주문 상세 조회용
CREATE TABLE orders_by_id (
    order_id UUID PRIMARY KEY,
    customer_id UUID,
    order_date timestamp,
    total_amount decimal,
    status text,
    items list<text>
);
```

**Step-by-Step**:
1. 각 쿼리 패턴별 별도 테이블 생성
2. Partition Key: WHERE 절의 등호 조건 컬럼
3. Clustering Column: ORDER BY 및 범위 조건 컬럼
4. 중복 데이터 허용 (반정규화)

### 3. Physical Design (물리적 설계)

```cql
-- TTL 설정 (임시 데이터용)
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID,
    data text,
    created_at timestamp
) WITH ttl = 86400;  -- 24시간 자동 삭제

-- 컴팩션 전략 설정
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    event_type text,
    payload text,
    created_at timestamp
) WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'window_size_in_seconds': 3600
};

-- 압축 설정
CREATE TABLE logs (
    log_id UUID PRIMARY KEY,
    message text,
    level text,
    created_at timestamp
) WITH compression = {
    'sstable_compression': 'SnappyCompressor'
};
```

**Step-by-Step**:
1. 데이터 수명 주기 분석 → TTL 적용 여부 결정
2. 쓰기/읽기 비율에 따라 컴팩션 전략 선택
3. 데이터 타입 최적화 (text vs blob, list vs set)
4. 저장 공간 고려 → 압축 설정

### 4. RDBMS 마이그레이션 패턴

```sql
-- RDBMS (조인 사용)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```cql
-- Cassandra (반정규화)
CREATE TABLE customer_orders (
    customer_id UUID,
    order_id UUID,
    customer_name text,  -- 중복 저장
    order_date timestamp,
    total_amount decimal,
    PRIMARY KEY (customer_id, order_date, order_id)
);

-- 별도 테이블: 고객 정보 조회용
CREATE TABLE customers_by_id (
    customer_id UUID PRIMARY KEY,
    name text,
    email text,
    created_at timestamp
);
```

**Step-by-Step**:
1. 조인 패턴 식별
2. 각 엔티티를 별도 테이블로 분리
3. 자주 함께 조회되는 데이터는 중복 저장
4. 일관성 유지를 위한 업데이트 전략 수립

### 5. 모델 평가 및 개선

```cql
-- 핫스팟 확인 (nodetool stats 사용)
nodetool stats my_keyspace

-- 파티션 크기 확인
SELECT * FROM system_schema.tables 
WHERE keyspace_name = 'my_app';
```

**Step-by-Step**:
1. 파티션 크기 모니터링 (권장: 100MB 이하)
2. 읽기/쓰기 레이턴시 측정
3. 컴팩션 부하 확인
4. 필요시 테이블 분할 또는 병합

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | nodetool로 메트릭 확인 | `nodetool stats keyspace_name` |
| `search_files` | 쿼리 패턴 검색 | `grep -r "WHERE" *.sql` |
| `read_file` | 스키마 파일 분석 | schema migration 파일 읽기 |
| `edit_file` | 테이블 구조 수정 | DDL 변경 적용 |

---

## Anti-Patterns & Guardrails

❌ **조인 사용 시도** — Cassandra는 조인을 지원하지 않음 (반정규화로 해결)  
❌ **단일 파티션에 과도한 데이터** — 100MB 초과 시 성능 저하, 읽기 불가  
❌ **동일 Partition Key에 집중된 쓰기** — Hotspot 발생, 레플리카 과부하  
❌ **RDBMS 정규화 패턴 적용** — Cassandra는 반정규화 필수  
❌ **Clustering Column 순서 변경** — ALTER TABLE로 변경 불가, 재생성 필요  
❌ **대용량 list/set 사용** — 10만 항목 초과 시 성능 저하  

⚠️ **Partition Key 카디널리티 낮음** — 데이터 편중 발생, 분산 실패  
⚠️ **TTL과 Compaction 전략 충돌** — TTL 적용 시 TWCS 권장  
⚠️ **Materialized Views 남용** — 쓰기 성능 저하 + 저장 공간 2배  

---

## Best Practices

1. **쿼리 패턴 먼저 분석** — 모든 읽기 쿼리를 목록화
2. **Partition Key는 고카디널리티** — 데이터 분산 보장
3. **Clustering Column은 정렬 목적에 맞게** — ORDER BY와 일치
4. **반정규화 적극 활용** — 중복 데이터 허용, 읽기 성능 우선
5. **파티션 크기 모니터링** — 100MB 이하 유지 권장
6. **Hotspot 방지** — Partition Key에 시퀀스/타임스탬프 추가 고려
7. **테이블 분리 전략** — 각 쿼리 패턴별 별도 테이블 생성

---

## References

- [Introduction to Data Modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/intro.html)
- [Conceptual Data Modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/data-modeling_conceptual.html)
- [RDBMS Design Patterns](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/data-modeling_rdbms.html)
- [Defining Application Queries](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/data-modeling_queries.html)
- [Logical Data Modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/data-modeling_logical.html)
- [Physical Data Modeling](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/data-modeling_physical.html)
- [Evaluating and Refining Models](https://cassandra.apache.org/doc/latest/cassandra/developing/data-modeling/data-modeling_refining.html)
