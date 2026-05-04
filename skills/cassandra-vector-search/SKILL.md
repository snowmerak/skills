---
name: cassandra-vector-search
description: Apache Cassandra 5.0+의 Vector Search 기능을 다룹니다. 벡터 데이터 타입, 유사도 검색 쿼리, SAI(Storage Attached Index)와의 통합, 임베딩 저장 및 조회 패턴 등을 학습합니다. AI/ML 애플리케이션과의 통합에 최적화되어 있습니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, vector-search, embedding, ai, ml, similarity-search, sai]
---

# Apache Cassandra Vector Search

## Overview

Cassandra 5.0+는 벡터 유사도 검색 기능을 native로 지원합니다. SAI(Storage Attached Index)와 통합되어 임베딩 기반 검색을 효율적으로 수행할 수 있습니다. AI/ML 애플리케이션에서 임베딩 저장 및 유사도 검색에 최적화되었습니다.

**핵심 철학**: 분산 데이터베이스에서의 효율적인 벡터 검색, 실시간 임베딩 업데이트 지원.

---

## SOP: Step-by-Step Procedures

### 1. Vector 데이터 타입 이해

```cql
-- vector 데이터 타입 정의
CREATE TABLE embeddings (
    id UUID PRIMARY KEY,
    content text,
    embedding vector<float, 768>,  -- 768차원 벡터
    metadata map<text, text>,
    created_at timestamp
);

-- 차원 확인
DESCRIBE TABLE embeddings;
```

**Vector 타입 특징**:
- `vector<float, N>` 또는 `vector<double, N>` 사용
- N은 차원 수 (예: 768, 1536, 3072)
- 고정된 차원 수만 허용 (동적 변경 불가)
- 최대 차원 수는 시스템 제한 있음

### 2. 임베딩 저장

```cql
-- 단일 임베딩 삽입
INSERT INTO embeddings (id, content, embedding, metadata, created_at)
VALUES (
    uuid(),
    'This is a sample document',
    {0.1, 0.2, 0.3, ..., 0.768},  -- 실제 임베딩 값
    {'source': 'web', 'language': 'en'},
    now()
);

-- 배치 삽입 (UNLOGGED BATCH)
BEGIN UNLOGGED BATCH
    INSERT INTO embeddings (id, content, embedding) VALUES (uuid(), 'doc1', {...});
    INSERT INTO embeddings (id, content, embedding) VALUES (uuid(), 'doc2', {...});
    INSERT INTO embeddings (id, content, embedding) VALUES (uuid(), 'doc3', {...});
APPLY BATCH;
```

**Step-by-Step**:
1. 임베딩 모델 출력 차원에 맞는 vector 타입 정의
2. 원본 콘텐츠와 메타데이터 함께 저장
3. `UNLOGGED BATCH`로 배치 삽입 (성능 최적화)
4. 타임스탬프로 최신 데이터 추적

### 3. 유사도 검색 쿼리

```cql
-- 코사인 유사도 기반 검색
SELECT id, content, metadata, cosine_similarity(embedding, {0.1, 0.2, ...}) AS similarity
FROM embeddings
ORDER BY similarity DESC
LIMIT 10;

-- 내적(dot product) 기반 검색
SELECT id, content, dot_product(embedding, {0.1, 0.2, ...}) AS score
FROM embeddings
ORDER BY score DESC
LIMIT 10;

-- 유클리드 거리 기반 검색
SELECT id, content, euclidean_distance(embedding, {0.1, 0.2, ...}) AS distance
FROM embeddings
ORDER BY distance ASC
LIMIT 10;
```

**Step-by-Step**:
1. 쿼리 임베딩 생성 (동일 모델 사용)
2. 유사도 함수 선택 (cosine_similarity 권장)
3. `ORDER BY`로 정렬, `LIMIT`으로 결과 수 제한
4. 임계값 필터링 필요시 WHERE 절 추가

### 4. SAI 인덱싱

```cql
-- 기본 SAI 인덱스 (자동 생성)
CREATE INDEX ON embeddings (embedding);

-- 인덱스 옵션 설정
CREATE CUSTOM INDEX embedding_idx ON embeddings (embedding)
USING 'StorageAttachedIndex'
WITH OPTIONS = {
    'similarity_function': 'cosine',
    'max_neighbors': 32,
    'build_on_flush': true
};
```

**SAI 인덱스 옵션**:
- `similarity_function`: 유사도 함수 (cosine, dot_product, euclidean)
- `max_neighbors`: KNN 검색 시 탐색할 이웃 수 (정확도/속도 트레이드오프)
- `build_on_flush`: SSTable flush 시 인덱스 빌드

**Step-by-Step**:
1. 기본 인덱스로 시작 (자동 SAI 생성)
2. 성능 테스트 후 옵션 조정
3. `max_neighbors` 증가 → 정확도 ↑, 속도 ↓
4. `build_on_flush` 활성화로 실시간 인덱싱

### 5. 임베딩 업데이트 및 삭제

```cql
-- 임베딩 업데이트
UPDATE embeddings
SET embedding = {0.2, 0.3, ...}, metadata = {'version': '2'}
WHERE id = <uuid_value>;

-- 조건부 업데이트 (CAS)
UPDATE embeddings
SET embedding = {0.2, 0.3, ...}
WHERE id = <uuid_value>
IF embedding = {0.1, 0.2, ...};

-- 임베딩 삭제
DELETE FROM embeddings WHERE id = <uuid_value>;

-- 만료된 임베딩 정리 (TTL 활용)
ALTER TABLE embeddings ADD expired_at timestamp;
UPDATE embeddings SET expired_at = now() + TTL(86400) WHERE id = <uuid>;
```

**Step-by-Step**:
1. `UPDATE`로 기존 임베딩 재작성
2. CAS(`IF`)로 동시성 제어
3. TTL로 만료 데이터 자동 정리 고려
4. 삭제 시 인덱스 자동 업데이트

### 6. 성능 튜닝

```yaml
# cassandra.yaml 벡터 검색 관련 설정
sai_index_memory_in_mb: 512  # SAI 인덱스 메모리 할당
concurrent_compactors: 4     # 컴팩션 병렬 처리 증가
```

**Step-by-Step**:
1. `sai_index_memory_in_mb` 벡터 수에 맞게 조정
2. 컴팩션 스레드 수 증가 (벡터 인덱스 오버헤드 고려)
3. 메모리 사용량 모니터링
4. 쿼리 레이턴시 측정 및 최적화

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | cqlsh로 벡터 쿼리 실행 | `cqlsh -e "SELECT ..."` |
| `search_files` | 임베딩 관련 코드 검색 | `grep -r "cosine_similarity" *.py` |
| `read_file` | 설정 파일 분석 | cassandra.yaml SAI 옵션 확인 |
| `edit_file` | 인덱스 설정 변경 | SAI 옵션 수정 적용 |

---

## Anti-Patterns & Guardrails

❌ **차원 수 동적 변경** — vector 타입 차원은 고정, 테이블 재생성 필요  
❌ **LIMIT 없이 유사도 검색** — 전체 스캔, 성능 치명적 저하  
❌ **대용량 배치 삽입 (10만+)** — 메모리 오버플로우 위험  
❌ **SAI 인덱스 미사용** — 벡터 검색 시 전체 테이블 스캔  
❌ **동일 Partition에 과도한 벡터 쓰기** — Hotspot 발생  

⚠️ **벡터 차원 수 불일치** — 삽입 실패, 데이터 무결성 손상  
⚠️ **max_neighbors过小** — 정확도 저하, 관련 결과 누락  
⚠️ **임베딩 모델 변경 시 재인덱싱 필요** — 기존 인덱스 무효화  

---

## Best Practices

1. **cosine_similarity 기본 사용** — 방향 유사도 측정 최적
2. **SAI 인덱스 필수 적용** — KNN 검색 성능 극대화
3. **LIMIT 항상 설정** — 결과셋 크기 제어
4. **메타데이터 함께 저장** — 검색 결과 컨텍스트 제공
5. **TTL로 만료 데이터 관리** — 자동 정리 파이프라인 구축
6. **max_neighbors 튜닝** — 정확도/속도 균형 찾기
7. **임베딩 모델 버전 추적** — 재인덱싱 시기 결정

---

## References

- [Vector Search Overview](https://cassandra.apache.org/doc/latest/cassandra/vector-search/overview.html)
- [Vector Data Type Reference](https://cassandra.apache.org/doc/latest/cassandra/reference/vector-data-type.html)
- [SAI Virtual Table Indexes](https://cassandra.apache.org/doc/latest/cassandra/reference/sai-virtual-table-indexes.html)
- [CQL Functions (cosine_similarity, dot_product)](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/functions.html)
