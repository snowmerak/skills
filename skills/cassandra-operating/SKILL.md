---
name: cassandra-operating
description: Apache Cassandra 클러스터 운영 전반을 다룹니다. 백업/복구, 리페어(Repair), 컴팩션(Compaction), 모니터링, 보안 설정, 하드웨어 가이드라인, 트랜지언트 레플리케이션 등 운영 실무 작업을 수행할 때 사용합니다.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: cassandra
  tags: [cassandra, operations, backup, repair, compaction, monitoring, security]
---

# Apache Cassandra Operating

## Overview

Cassandra 클러스터 운영은 분산 시스템의 특성상 특별한 주의가 필요합니다. 백업, 리페어, 컴팩션, 모니터링 등 핵심 운영 작업을 체계적으로 수행해야 안정성을 보장할 수 있습니다.

**핵심 원칙**: 자동화 우선, 모니터링 상시, 예방적 유지보수.

---

## SOP: Step-by-Step Procedures

### 1. 백업 및 복구 (Backups & Restore)

```bash
# 스냅샷 생성
nodetool snapshot -t backup_20240101 my_keyspace

# 스냅샷 목록 확인
nodetool listsnapshots

# sstableloader로 다른 노드에 데이터 복원
sstableloader -d <target_node_ip> /path/to/sstables

# 스냅샷 삭제 (복구 후)
nodetool clearsnapshot -t backup_20240101
```

**Step-by-Step**:
1. `nodetool snapshot`으로 현재 상태 스냅샷 생성
2. 스냅샷 디렉토리 백업 (S3/GCS 등 외부 저장소)
3. 복구 시 `sstableloader` 또는 `sstablerepairedex` 사용
4. 복구 완료 후 스냅샷 정리

### 2. 리페어 (Repair)

```bash
# 전체 클러스터 리페어
nodetool repair

# 특정 키스페이스 리페어
nodetool repair my_keyspace

# 아카이브 리페어 (Auto Repair 권장)
# cassandra.yaml에서 auto_snapshot 설정
auto_snapshot: false  # 리페어 시 스냅샷 비활성화
```

**Step-by-Step**:
1. 정기적 리페어 스케줄링 (일일/주간)
2. `nodetool repair -pr`로 파티션 레인저 사용 권장
3. Auto Repair 설정으로 자동화 고려
4. 리페어 중 모니터링 (CPU, 네트워크 부하 확인)

### 3. 컴팩션 관리 (Compaction)

```yaml
# cassandra.yaml 컴팩션 전략 설정
compaction_throughput_mb_per_sec: 16  # 초당 MB 단위
compaction_large_partition_warning_threshold_mb: 100

# SizeTiered Compaction (쓰기 중심)
STCS:
  class: org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy
  max_threshold: 32
  min_threshold: 4

# Leveled Compaction (읽기 중심)
LCS:
  class: org.apache.cassandra.db.compaction.LeveledCompactionStrategy
```

**Step-by-Step**:
1. 워크로드 유형에 따라 컴팩션 전략 선택
   - STCS: 쓰기 빈도 높음
   - LCS: 읽기 빈도 높음
   - TWCS: 타임스탬프 기반 데이터 (로그, 이벤트)
2. `compaction_throughput_mb_per_sec` 튜닝
3. 컴팩션 모니터링: `nodetool compactionstats`

### 4. 모니터링 메트릭

```bash
# 노드 상태 확인
nodetool status

# 메트릭 확인
nodetool stats my_keyspace
nodetool tpstats  # 스레드 풀 대기 상태
nodetool tablestats my_keyspace my_table

# GC 정보
nodetool gcstats

# 커넥션 정보
nodetool netstats
```

**Step-by-Step**:
1. `nodetool status`로 클러스터 건강 상태 확인
2. `nodetool tpstats`로 스레드 풀 대기 시간 모니터링
3. `nodetool tablestats`로 테이블별 읽기/쓰기 레이턴시 추적
4. GC 로그 분석으로 메모리 문제 진단

### 5. 보안 설정 (Security)

```cql
-- 인증 활성화 (cassandra.yaml)
authenticator: PasswordAuthenticator

-- 인가 활성화
authorizer: CassandraAuthorizer

-- SSL/TLS 설정
client_encryption_options:
  enabled: true
  optional: false
  keystore: /path/to/keystore.jks
  keystore_password: <password>

-- RBAC 권한 부여
CREATE USER admin WITH PASSWORD 'secure_password' SUPERUSER;
GRANT SELECT ON KEYSPACE my_app TO app_user;
```

**Step-by-Step**:
1. `PasswordAuthenticator` 활성화
2. `CassandraAuthorizer`로 RBAC 설정
3. SSL/TLS로 노드 간/클라이언트 암호화
4. 최소 권한 원칙 적용 (GRANT/REVOKE)

### 6. 하드웨어 가이드라인

```yaml
# 권장 사양
CPU: 8+ 코어 (컴팩션에 많은 CPU 필요)
RAM: 32GB+ (메모리 풀 할당 고려)
Disk: SSD 필수 (SSTable I/O 성능 중요)
Network: 1Gbps 이상 (노드 간 통신)

# 메모리 설정 (jvm-server.options)
# Heap size는 6-8GB 권장 (초과 시 성능 저하)
-Xms6g
-Xmx6g
```

**Step-by-Step**:
1. SSD 디스크 필수 사용 (HDD 비권장)
2. RAM은 메타데이터 저장용, Heap은 6-8GB로 제한
3. 네트워크 대역폭 확보 (노드 간 스트리밍 고려)
4. CPU 코어 수 확인 (컴팩션 병렬 처리)

### 7. 트랜지언트 레플리케이션

```bash
# 노드 추가 시 자동 복제
# cassandra.yaml에서 활성화
auto_bootstrap: true

# 수동 트랜지언트 리페어
nodetool repair -pr -t <new_node_ip>
```

**Step-by-Step**:
1. 새 노드 추가 시 `auto_bootstrap: true` 확인
2. 노드 합류 후 자동 리페어 실행
3. 데이터 동기화 완료 확인 (`nodetool status`)
4. 트랜지언트 데이터 정리

---

## Tool Integration

| 도구 | 사용 목적 | 예시 |
|------|-----------|------|
| `run_command` | nodetool, sstableloader 실행 | `nodetool repair my_keyspace` |
| `search_files` | 로그 파일 패턴 검색 | `grep -r "ERROR" system.log` |
| `read_file` | 설정 파일 분석 | cassandra.yaml 확인 |
| `edit_file` | 설정 변경 적용 | jvm 옵션 수정 |

---

## Anti-Patterns & Guardrails

❌ **리페어 생략** — 데이터 불일치 누적, 데이터 손실 위험  
❌ **HDD 사용** — SSTable I/O 성능 치명적 저하  
❌ **Heap 8GB 초과** — GC 지연 시간 증가, 서비스 중단 위험  
❌ **스냅샷 미정리** — 디스크 공간 고갈  
❌ **모니터링 부재** — 문제 조기 발견 불가  
❌ **SSL/TLS 비활성화** — 데이터 유출 위험 (프로덕션)  

⚠️ **대규모 리페어 시 네트워크 포화** — 시간대 분리 또는 분산 실행  
⚠️ **컴팩션 중 쓰기 지연** — 스로틀링 설정 권장  
⚠️ **GC 휴지 시간 초과** — Heap 크기 조정 필요  

---

## Best Practices

1. **정기적 리페어 스케줄링** — 일일 또는 주간 자동화
2. **모니터링 상시 유지** — 메트릭 수집 및 알림 설정
3. **SSD 디스크 필수** — HDD 사용 금지
4. **Heap 6-8GB로 제한** — 초과 시 성능 저하
5. **SSL/TLS 활성화** — 프로덕션 환경 필수
6. **스냅샷 자동 정리** — 디스크 공간 관리
7. **Auto Repair 고려** — 수동 리페어 오버헤드 감소

---

## References

- [Backups](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/backups.html)
- [Repair](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/repair.html)
- [Compaction](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/compaction/index.html)
- [Monitoring Metrics](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/metrics.html)
- [Security](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/security.html)
- [Hardware Guidelines](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/hardware.html)
- [Auto Repair](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/auto_repair.html)
