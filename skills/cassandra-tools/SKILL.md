---
name: cassandra-tools
description: Apache Cassandra의 명령줄 도구집을 다룹니다. cqlsh(CQL 셸), nodetool(클러스터 관리), SSTable tools(SSTable 처리), cassandra-stress(성능 테스트) 등 모든 공식 도구의 사용법을 학습합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, cqlsh, nodetool, sstable, cassandra-stress, cli-tools]
---

# Apache Cassandra Tools

## Overview

Cassandra는 클러스터 관리, 데이터 조작, 성능 테스트를 위한 다양한 명령줄 도구를 제공합니다. 이 스킬은 cqlsh, nodetool, SSTable tools, cassandra-stress의 핵심 사용법을 다룹니다.

**핵심 원칙**: 각 도구의 전문 분야를 이해하고 적절한 상황에 활용합니다.

---

## SOP: Step-by-Step Procedures

### 1. cqlsh — CQL Shell

```bash
# 기본 연결
cqlsh <host> <port> -u <username> -p <password>

# SSL 연결
cqlsh --ssl <host>

# 스크립트 실행
cqlsh -f migration.sql

# CSV 출력 형식
cqlsh --format=csv -e "SELECT * FROM users LIMIT 10;"

# Python 스크립트로 cqlsh 자동화
echo "DESCRIBE KEYSPACES;" | cqlsh localhost
```

**cqlsh 설정 파일**: `~/.cassandra/cqlshrc`

```ini
[connection]
hostname = localhost
port = 9042
factory = cqlshlib.cql.connection_factory

[auth]
username = admin
password = secure_password

[display]
paging = true
page_size = 100
```

**Step-by-Step**:
1. 연결 설정 확인 (호스트, 포트, 인증)
2. `-f` 옵션으로 스크립트 파일 실행
3. `--format=csv/json`로 출력 형식 지정
4. 자동완성 활성화 (기본 제공)

### 2. nodetool — 클러스터 관리

```bash
# 클러스터 상태 확인
nodetool status
nodetool status -p  # 포트 정보 포함

# 노드 메트릭
nodetool stats my_keyspace
nodetool tablestats my_table
nodetool tpstats    # 스레드 풀 대기 상태

# 컴팩션 관리
nodetool compactionstats      # 진행 중인 컴팩션 확인
nodetool compact              # 수동 컴팩션 트리거
nodetool compactionhistory    # 컴팩션 이력

# 스냅샷 관리
nodetool snapshot -t backup_20240101 my_keyspace
nodetool listsnapshots
nodetool clearsnapshot -t backup_20240101

# 리페어
nodetool repair my_keyspace
nodetool repair -pr           # 파티션 레인저 사용
nodetool repair -full         # 전체 클러스터 리페어

# 노드 제어
nodetool drain                # 데이터 이동 후 노드 종료 준비
nodetool stopdaemon           # nodetool 서비스 중지
nodetool startdaemon          # nodetool 서비스 시작

# 설정 확인/변경 (동적)
nodetool getcompactionthroughput
nodetool setcompactionthroughput 16
nodetool getlogginglevels
```

**Step-by-Step**:
1. `nodetool status`로 클러스터 건강 상태 확인
2. `nodetool tpstats`로 스레드 풀 대기 시간 모니터링
3. `nodetool compactionstats`로 컴팩션 부하 추적
4. 동적 설정 변경으로 성능 튜닝

### 3. SSTable Tools — SSTable 처리

```bash
# SSTable to CSV 변환
sstable2csv /path/to/sstables > data.csv

# CSV to SSTable 변환
csv2sstable -p <partition_key_index> data.csv my_table

# SSTable 구조 분석
sstabledump /path/to/sstable/my_table-abc123.db

# SSTable 디버깅
sstableverify keyspace table /path/to/sstables

# SSTable 메타데이터 확인
sstablemetadata /path/to/sstable/my_table-abc123.db

# SSTable 분할/병합
sstablerepairedex  # repaired 데이터 추출
sstablelevelize    # leveled compaction 강제
```

**Step-by-Step**:
1. `sstable2csv`로 데이터 내보내기
2. `csv2sstable`로 데이터 가져오기 (partition key 인덱스 필수)
3. `sstabledump`로 SSTable 구조 분석
4. `sstableverify`로 무결성 확인

### 4. cassandra-stress — 성능 테스트

```bash
# 기본 쓰기 테스트
cassandra-stress write n=100000 -mode cql3 user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop seq=1..100000 -rate threads=10

# 읽기 테스트
cassandra-stress read n=100000 -mode cql3 user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop uniform=1..100000 -rate threads=10

# 혼합 워크로드 (70% 읽기, 30% 쓰기)
cassandra-stress mixed n=1000000 -mode cql3 user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop seq=1..1000000 -rate threads=50 \
  -ops '{"write":1,"select":3}'

# 사용자 정의 쿼리 사용
cassandra-stress write n=10000 -mode cql3 native user=admin password=password \
  -schema "keyspace=test table=users replicas=1" \
  -pop seq=1..10000 \
  -query "INSERT INTO users (id, name) VALUES (?, ?)" \
  -rate threads=20
```

**Step-by-Step**:
1. 테스트 목적 정의 (쓰기/읽기/혼합)
2. 데이터 크기(n) 및 스레드 수(threads) 설정
3. 워크로드 패턴(population) 선택 (seq/uniform/gaussian)
4. 결과 분석: 레이턴시, TPS, 에러율

### 5. sstableloader — 대량 데이터 로드

```bash
# 다른 노드로 SSTable 복사
sstableloader -d <target_node_ip> /path/to/sstables

# 인증 포함
sstableloader -u admin -p password -d <target_node_ip> /path/to/sstables

# SSL 사용
sstableloader --ssl -d <target_node_ip> /path/to/sstables

# 스로틀링 설정 (네트워크 대역폭 제한)
sstableloader -t 100 -d <target_node_ip> /path/to/sstables
```

**Step-by-Step**:
1. SSTable 디렉토리 준비
2. 대상 노드 IP 지정
3. 인증/SSL 설정 확인
4. 네트워크 대역폭 고려하여 스로틀링 적용

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | 모든 Cassandra CLI 도구 실행 | `nodetool status`, `cqlsh -f script.sql` |
| `search_files` | 로그/설정 파일에서 패턴 검색 | `grep "ERROR" system.log` |
| `read_file` | 스크립트/설정 파일 읽기 | migration.sql, cqlshrc 분석 |
| `edit_file` | 설정 파일 수정 | nodetool 동적 설정 변경 기록 |

---

## Anti-Patterns & Guardrails

❌ **nodetool repair -full 빈번 실행** — 클러스터 과부하, 성능 저하  
❌ **sstabledump 대용량 SSTable 사용** — 메모리 고갈 위험  
❌ **cassandra-stress 테스트 후 정리 안함** — 테스트 데이터 누수  
❌ **sstableloader 스로틀링 없이 실행** — 네트워크 포화, 서비스 영향  
❌ **nodetool drain 없이 노드 제거** — 데이터 손실 위험  

⚠️ **cqlsh 대량 INSERT 비권장** — 배치 크기 제한 (1000 이하 권장)  
⚠️ **nodetool compact 빈번 실행** — 컴팩션 오버헤드 증가  
⚠️ **sstable2csv 대용량 데이터 사용** — 디스크 공간 고려  

---

## Best Practices

1. **nodetool status 상시 확인** — 클러스터 건강 상태 모니터링
2. **cqlsh 스크립트 버전 관리** — migration 파일로 관리
3. **cassandra-stress 정기적 테스트** — 성능 베이스라인 유지
4. **sstabledump 디버깅용 활용** — SSTable 구조 분석
5. **sstableloader 스로틀링 필수** — 네트워크 대역폭 고려
6. **nodetool 동적 설정 활용** — 재시작 없이 튜닝 가능
7. **도구 출력 자동화** — 스크립트로 모니터링 파이프라인 구축

---

## References

- [cqlsh Documentation](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/cqlsh.html)
- [nodetool Reference](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/nodetool/nodetool.html)
- [SSTable Tools](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/sstable/index.html)
- [cassandra-stress](https://cassandra.apache.org/doc/latest/cassandra/managing/tools/cassandra_stress.adoc)
