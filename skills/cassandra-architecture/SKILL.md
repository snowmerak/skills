---
name: cassandra-architecture
description: Apache Cassandra의 내부 아키텍처와 동작 원리를 다룹니다. Dynamo 논문 기반 설계, Storage Engine 구조(SSTable/Memtable/CommitLog), 일관성 보장 메커니즘(Consistency Levels), 노드 간 메시징 및 스트리밍, Snitches 등을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, architecture, storage-engine, consistency, dynamo, snitch]
---

# Apache Cassandra Architecture

## Overview

Cassandra는 Amazon의 Dynamo 논문과 Google의 Bigtable 논문을 결합한 분산 NoSQL 데이터베이스입니다. 높은 가용성, 선형 확장성, 다중 데이터센터 지원을 핵심 설계 목표로 합니다.

**핵심 철학**: 단일 장애점 제거(Single Point of Failure), 파티셔닝 기반 분산, 최종 일관성(Eventual Consistency) 허용.

---

## SOP: Step-by-Step Procedures

### 1. Dynamo 기반 설계 이해

```
Ring 구조 (가상 노드):
┌─────────────────────────────────────────────┐
│  vnode  │  vnode  │  vnode  │  vnode       │
│  N3     │  N1     │  N4     │  N2          │
└─────────────────────────────────────────────┘
         ↑           ↑           ↑
      Token Range  Token Range  Token Range
```

**Step-by-Step**:
1. Ring 구조 이해: 노드들이 원형으로 배치
2. Virtual Nodes(vnode)로 데이터 분산 최적화 (기본값: 256)
3. Token Range로 데이터 파티셔닝
4. Consistent Hashing으로 부하 균형

### 2. Storage Engine 구조

```
Write Path:
Client → CommitLog → Memtable → SSTable

Read Path:
Client → Query Cache → Partition Cache → Memtable + SSTables
```

**Component Breakdown**:
- **CommitLog**: 크래시 리커버리용 지속 로그 (append-only)
- **Memtable**: 메모리 내 쓰기 버퍼 (정렬된 구조)
- **SSTable**: 디스크에 저장된 읽기 전용 정적 파일
- **Query Cache**: 읽기 결과 캐싱 (Row cache)
- **Partition Cache**: 파티션 메타데이터 캐싱

**Step-by-Step**:
1. 쓰기: CommitLog → Memtable 순서로 기록
2. Memtable 임계치 도달 시 flush → SSTable 생성
3. 읽기: Cache → Memtable → SSTable 순차적 조회
4. Compaction으로 여러 SSTable 병합

### 3. 일관성 보장 (Consistency Levels)

```cql
-- 다양한 Consistency Level 사용 예시

-- 가장 강한 일관성 (모든 레플리카 확인)
INSERT INTO users (id, name) VALUES (1, 'John') USING CONSISTENCY ALL;

-- 균형 잡힌 일관성 (QUORUM 권장)
SELECT * FROM users WHERE id = 1 USING CONSISTENCY QUORUM;

-- 빠른 읽기 (ONE: 최소 한 개만 확인)
SELECT * FROM users WHERE id = 1 USING CONSISTENCY ONE;
```

**Consistency Level 비교**:

| CL | 쓰기 확인 | 읽기 확인 | 설명 |
|----|-----------|-----------|------|
| ANY | X | X | Hinted Handoff 허용, 가장 빠름 |
| ONE | 1 | 1 | 최소 한 개 노드 확인 |
| QUORUM | N/2+1 | N/2+1 | 과반수 확인 (권장) |
| LOCAL_QUORUM | DC: N/2+1 | DC: N/2+1 | 동일 DC만 확인 |
| ALL | N | N | 모든 레플리카 확인 |
| SERIAL | N/2+1 | N/2+1 | 조건부 업데이트용 |

**Step-by-Step**:
1. 워크로드에 맞는 CL 선택
   - 읽기 중심: QUORUM 또는 LOCAL_QUORUM
   - 쓰기 중심: ONE 또는 QUORUM
   - 금융/중요 데이터: ALL 또는 SERIAL
2. Replication Factor과 CL의 관계 이해 (RF > CL 필요)
3. 가용성 vs 일관성 트레이드오프 고려

### 4. Internode Messaging

```yaml
# cassandra.yaml 메시징 설정
native_transport_port: 9042  # CQL 클라이언트 포트
ssl_storage_port: 7001       # SSL 사용 시 노드 간 통신
storage_port: 7000           # 노드 간 통신 (비SSL)
```

**Step-by-Step**:
1. 노드 간 통신 포트 확인 (7000/7001)
2. CQL 클라이언트 포트 확인 (9042)
3. SSL 설정 시 포트 분리
4. 방화벽 규칙으로 포트 개방

### 5. Streaming 메커니즘

```bash
# 스트리밍 상태 확인
nodetool netstats

# 수동 스트리밍 트리거 (노드 추가 시 자동 실행)
nodetool move <token_value>
```

**Step-by-Step**:
1. 노드 추가/제거 시 데이터 재배치 자동 시작
2. `nodetool netstats`로 스트리밍 진행률 확인
3. 대역폭 제한 필요시 `stream_throughput_outbound_megabits_per_sec` 설정
4. 스트리밍 완료 후 클러스터 상태 확인

### 6. Snitches 이해 및 설정

```yaml
# cassandra.yaml Snitch 설정
endpoint_snitch: GossipingPropertyFileSnitch
```

**Snitch 유형**:

| Snitch | 사용처 | 설명 |
|--------|--------|------|
| SimpleSnitch | 단일 DC | 순환 방식으로 레플리카 배치 |
| PropertyFileSnitch | 테스트/소규모 | 설정 파일로 직접 정의 |
| GossipingPropertyFileSnitch | 프로덕션 (단일 DC) | 기본값, gossip 프로토콜 사용 |
| EC2Snitch | AWS | AWS API로 리전/가용영역 감지 |
| EC2MultiRegionSnitch | AWS 다중 리전 |跨区域 레플리케이션 지원 |
| GCESnitch | Google Cloud | GCP 프로젝트/존 감지 |

**Step-by-Step**:
1. 클라우드 환경에 맞는 Snitch 선택
2. `cassandra-rackdc.properties`로 라크/데이터센터 설정
3. 다중 DC 환경에서는 NetworkTopologyStrategy와 함께 사용
4. Snitch 변경 시 클러스터 재시작 필요

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | nodetool로 아키텍처 상태 확인 | `nodetool status -p` |
| `search_files` | 설정 파일에서 Snitch/CL 패턴 검색 | `grep "endpoint_snitch" *.yaml` |
| `read_file` | cassandra.yaml 분석 | 아키텍처 관련 설정 확인 |
| `edit_file` | Snitch/CL 설정 변경 | endpoint_snitch 수정 |

---

## Anti-Patterns & Guardrails

❌ **SimpleSnitch 다중 DC 환경 사용** — 데이터센터 간 레플리카 분산 실패  
❌ **ALL Consistency Level 남용** — 쓰기 레이턴시 급증, 가용성 저하  
❌ **Replication Factor < Consistency Level** — 쓰기 실패 발생  
❌ **vnode 비활성화** — 데이터 불균형, 재배치 어려움  
❌ **CommitLog 디스크 부족** — 쓰기 불가, 서비스 중단  

⚠️ **Snitch 변경 시 클러스터 재시작 필요** — 계획된 유지보수 시간 활용  
⚠️ **CL 변경 시 레플리카 수 확인 필수** — RF < CL이면 오류 발생  
⚠️ **스트리밍 중 노드 제거 금지** — 데이터 손실 위험  

---

## Best Practices

1. **vnode 활성화** (기본값) — 데이터 분산 최적화
2. **QUORUM Consistency Level 권장** — 일관성/가용성 균형
3. **RF ≥ 3** — 내결함성 보장
4. **클라우드 Snitch 사용** — AWS/GCP/Azure 전용 Snitch 선택
5. **모니터링 상시 유지** — 스트리밍, 컴팩션, GC 상태 추적
6. **CommitLog 별도 디스크** — I/O 분리 권장
7. **메모리 풀 적정 크기 설정** — Heap 6-8GB, 메타데이터용 여유 확보

---

## References

- [Architecture Overview](https://cassandra.apache.org/doc/latest/cassandra/architecture/overview.html)
- [Dynamo Design](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
- [Storage Engine](https://cassandra.apache.org/doc/latest/cassandra/architecture/storage-engine.html)
- [Guarantees (Consistency)](https://cassandra.apache.org/doc/latest/cassandra/architecture/guarantees.html)
- [Improved Internode Messaging](https://cassandra.apache.org/doc/latest/cassandra/architecture/messaging.html)
- [Improved Streaming](https://cassandra.apache.org/doc/latest/cassandra/architecture/streaming.html)
- [Snitches](https://cassandra.apache.org/doc/latest/cassandra/architecture/snitch.adoc)
